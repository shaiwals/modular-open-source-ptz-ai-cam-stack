# Web Dashboard, Media Pipeline, and YOLO Tracking Bottlenecks for the GA8ED PTZ AI Camera

## Overview

This document describes the evolution of the GA8ED PTZ AI camera’s end‑to‑end web and media pipeline, from a simple YOLO `predict` loop to the current multi‑threaded `track`‑based design using ByteTrack, and documents key bottlenecks and fixes. It focuses on three areas: the FastAPI/MediaMTX/Caddy stack that powers the dashboard, the Jetson-side GStreamer/YOLO pipeline, and the changes to tracking and threading that stabilized performance.

The goal is to give future contributors a complete picture of how the system serves video, runs inference, and handles tracking so they can reproduce, debug, or extend the pipeline without rediscovering these issues.

## Web and Networking Stack

### Dashboard, MediaMTX, and Caddy

The web UI is served by a FastAPI/Uvicorn app, which exposes a PTZ dashboard and metrics endpoint. MediaMTX runs on the Jetson and acts as the media server, publishing the camera stream over RTSP and WebRTC/WHEP for consumption by browsers.

A GStreamer “bridge” process on the Jetson forwards H.264 RTP from the AI pipeline into MediaMTX’s RTSP input (typically `rtsp://127.0.0.1:8554/ai_cam`). In production, the browser does not talk directly to the Jetson; instead, an EC2 instance runs Caddy as a reverse proxy and terminates TLS for a public domain such as `open-source-ai-camera.ga8ed.com`, forwarding requests to the Jetson’s MediaMTX endpoints.

This results in the following logical flow:

- Browser → HTTPS to EC2/Caddy → proxied to Jetson/MediaMTX for WebRTC/WHEP video.
- Browser → HTTPS to EC2/Caddy → proxied to Jetson/FastAPI for control and status APIs.

### Tailscale as a Diagnostic Tool

Tailscale is installed on the Jetson and used primarily as an **engineering and diagnostic tool** for secure SSH access and occasional direct dashboard testing, especially when debugging over LTE or during early bring‑up. It is **not** part of the normal end‑user request path in the final architecture; customers and reviewers access the camera via the public domain and Caddy, not through the Tailscale Tailnet.

Earlier design iterations explored Tailscale as a fleet‑wide access mechanism and discussed ACLs, node sharing, and Funnel for multi‑tenant neighborhood access, but those ideas are treated as future options rather than part of the current deployed pipeline. The production data path for the capstone implementation uses standard HTTPS + reverse proxying, leaving Tailscale as an admin‑only safety valve for SSH and low‑level debugging.

## MediaMTX and WebRTC Behavior

### Freezing and Buffer Growth

During extended use, the WebRTC stream occasionally froze after several minutes, requiring a page refresh. Debugging indicated that this behavior often results from jitter buffer buildup or missing keyframes, where the browser’s decoder no longer receives a clean I‑frame to resynchronize.

MediaMTX logs on the Jetson served as the main diagnostic source, indicating whether WebRTC sessions were closing or complaining about missing keyframes. Browser‑side tools like `chrome://webrtc-internals` were used to inspect jitter buffer delay and dropped frames.

### Encoder and MediaMTX Tuning

To improve stability, the Jetson’s GStreamer pipeline that feeds MediaMTX was tuned:

- H.264 encoder: `nvv4l2h264enc` (where available) or `x264enc` was configured with frequent keyframes and SPS/PPS insertion so browsers receive regular I‑frames suitable for recovery.
- Bitrate: The encoder bitrate was reduced (for example to around 2–4 Mbps at 720p) to avoid saturating uplinks and causing upstream congestion.

On the MediaMTX side, the `mediamtx.yml` configuration was adjusted:

- Increasing `readTimeout` and `writeTimeout` to tolerate brief jitter without dropping connections.
- Ensuring WebRTC UDP transport was enabled and correctly advertised via relevant `webrtc*` parameters so connections preferentially use UDP rather than TCP where possible.

Reverse proxy behavior via Caddy on EC2 was also checked to prevent premature timeouts on the WHEP signaling path.

## Initial YOLO Pipeline (Single AI Thread, `track` + `plot`)

### Structure and Bottlenecks

The earliest YOLO pipeline version used a two‑thread model:

- Capture thread: GStreamer `appsink0` pulled frames into CPU memory and published them as `FramePacket`s.
- AI thread: `_ai_loop` pulled the latest packet, ran `self._yolo_model.track(...)` (with fallback to `predict(...)`), plotted bounding boxes (`results.plot()`), encoded to JPEG via `cv2.imencode()`, and then updated `latest_jpeg` for the dashboard.

This design pushed all CPU‑heavy work—tracking, plotting, and JPEG encoding—into the same thread as GPU inference, resulting in roughly 12 FPS on the Jetson. The GPU spent substantial time idle while the ARM CPU performed overlay and compression.

### Track vs Predict in the First Version

In this earliest version, Ultralytics `track` used the default tracker (BoT‑SORT with a ReID network), which runs a second model on the CPU to provide appearance‑based re‑identification. That significantly increased per‑frame latency compared to `predict`, which only runs the detection network.

Switching from `track` to `predict` removed the CPU‑bound tracker overhead and slightly improved inference times, but did not fully resolve the FPS bottleneck because post‑processing still dominated the loop. Plotting and JPEG encoding remained the primary limiting factors.

## Introduction of Multi-Threaded Encoder

### Offloading Plotting and Encoding

To decouple AI and encoding, a third thread, `_encoder_loop`, was added, and a shared `_encode_latest` buffer guarded by `_encode_lock` was introduced. The AI thread became responsible only for inference, storing `(frame, results)` in `_encode_latest`, while the encoder thread handled:

- Constructing overlays with `results.plot()` if detections were present.
- Resizing the overlay to the configured stream resolution.
- Writing frames into a GStreamer‑backed `cv2.VideoWriter` that sends RTP packets to a local UDP port, from which a GStreamer bridge relays them into MediaMTX.

This reorganization relieved the AI loop from CPU plotting and encoding, allowing the GPU to be kept busier and unlocking significantly higher FPS with `predict` mode.

### Background Services and Bridge Process

The pipeline also launched two background services:

- MediaMTX RTSP/WebRTC server, started via `subprocess.Popen` using a defined binary path.
- A GStreamer `gst-launch-1.0` bridge that reads RTP from UDP port 5000 and pushes it into `rtsp://127.0.0.1:8554/ai_cam` on MediaMTX, initially using `protocols=tcp` and later optimized to `protocols=udp` to reduce CPU overhead on local loopback.

## Tracking and Thread Decoupling Improvements

### Decoupling Encoder from AI Results

Further analysis showed that even with a separate encoder thread, the pipeline still had hidden coupling: the encoder loop waited for fresh AI results before pushing out frames. This implicitly tied video FPS to AI FPS.

A subsequent refactor changed the architecture so that:

- The encoder loop pulls the latest raw frame directly from `_raw_latest` (protected by `_raw_lock`) on every iteration.
- The encoder uses the latest detection set from `_latest_detections` (protected by `_detections_lock`), even if those detections are a frame or two behind.
- The encoder never blocks on AI; if no new detections are available, it still streams video, just without updated overlays.

This decoupling allowed video output to run at near‑camera FPS (≈30 FPS) while AI inference ran at its own pace.

### Moving Away from `results.plot()`

The pipeline moved away from Ultralytics `results.plot()` in the encoder loop, replacing it with custom overlay methods that operate directly on the NumPy frame:

- `_draw_boxes` draws bounding boxes and labels in place, using OpenCV primitives and avoiding extra copies.
- `_draw_tracking_focus` highlights the tracked target while drawing non‑tracked objects more lightly to reduce clutter and drawing work.

These custom functions reduced Python‑level overhead, especially full‑frame blending and multiple `.copy()` calls.

## Predict vs Track: Stability, Throughput, and ByteTrack

### Predict Mode Characteristics

`predict` mode runs pure detection on each frame and outputs bounding boxes with class and confidence data but no temporal identity (no track IDs). This is lighter weight and more GPU‑bound but unstable for long‑term target following because entity IDs can’t be maintained across frames.

In the multi‑threaded pipeline, `predict` achieved higher FPS than the original `track` implementation because it avoided both the BoT‑SORT tracker and its ReID network as well as some of the heaviest drawing paths; however, it did not provide the stable ID semantics needed for PTZ lock‑on.

### Default Tracker (`track`) and Its Cost

Ultralytics’ default `track` uses BoT‑SORT with a ReID model, effectively stacking a second neural network on top of YOLO for appearance‑based tracking. On the Jetson, this raised per‑frame inference time from about 30–32 ms (pure detection) to roughly 100 ms or more when `track` was enabled with the default configuration.

This confirmed that the tracking stage, not YOLO detection itself, was responsible for much of the latency when using the default tracker.

### Final Tracker Choice: ByteTrack

To restore performance while preserving stable IDs, the final pipeline explicitly switches Ultralytics’ tracker to **ByteTrack** by calling:

```python
results = self._yolo_model.track(
    packet.frame,
    imgsz=self.settings.yolo_imgsz,
    conf=self.settings.yolo_confidence,
    persist=True,
    tracker="bytetrack.yaml",
    verbose=False,
)
```

ByteTrack removes the heavy ReID model and uses IOU + Kalman filtering on bounding box coordinates to track objects over time. This dramatically reduces CPU load compared to BoT‑SORT while still providing stable track IDs that are sufficient for PTZ control and UI lock‑on.

With ByteTrack in place and the encoder decoupled, `track` mode’s inference time returns close to the ~40 ms range seen in pure detection mode, while stream FPS remains locked near 24 FPS.

### Interpolation/Stride Experiment and Final Decision

An intermediate version added an `ai_stride` parameter and an `_interpolate` helper that used the tracker’s internal state to predict positions on skipped frames, running full YOLO only every Nth frame. This approach can boost apparent FPS in some architectures, but in this pipeline it did not substantially improve throughput once the encoder was decoupled and CPU hotspots were removed.

The final design therefore **removes** stride‑based interpolation entirely. ByteTrack runs on every frame, providing consistent track IDs, while the decoupled encoder ensures that video remains smooth even when inference time varies.

## CPU Bottlenecks and Fixes

### Full-Frame Alpha Blending

The `_draw_tracking_focus` implementation originally created a separate “faded” layer and blended it with the base frame using `cv2.addWeighted` on full 2560×1440 frames. This implied two full image copies and a per‑pixel blend operation on large matrices, all on the CPU.

The fix was to:

- Eliminate full‑frame alpha blending and multiple `.copy()` calls.
- Draw non‑tracked objects with thinner, darker outlines directly on the original frame, and reserve heavier highlighting for the tracked target only.

### GStreamer `videoconvert` in the Capture Pipeline

The original GStreamer capture pipeline performed `nvvidconv` to BGRx followed by a CPU `videoconvert` to BGR, carrying out expensive color conversion at high resolution on the CPU. This was removed from the pipeline string, leaving only `nvvidconv` to BGRx, and the alpha channel was stripped in Python using NumPy slicing.

Specifically:

- GStreamer pipeline now outputs `video/x-raw, format=BGRx`.
- `_sample_to_bgr_frame` maps the buffer into a `(H, W, 4)` array and slices `[:, :, :3]` to obtain BGR, copying it once.

### Redundant Resizing and Copies

The encoder loop originally resized frames unconditionally and made multiple `.copy()` calls in box‑drawing helpers. Fixes included:

- Only calling `cv2.resize` when the overlay dimensions differ from the target stream resolution.
- Copying the frame once in the encoder loop and then passing that same array by reference into `_draw_boxes` and `_draw_numbered_labels`, which mutate it in place.

### Localhost TCP Overhead in Bridge

The GStreamer bridge process initially configured `rtspclientsink` with `protocols=tcp`, forcing RTSP over TCP from `127.0.0.1` to MediaMTX. TCP adds unnecessary per‑packet overhead on local loopback where packet loss is negligible.

Changing `protocols=tcp` to `protocols=udp` in the bridge command significantly reduced CPU work for local streaming, improving overall responsiveness on the Jetson.

### Encoding Resolution and Bitrate

The Orin Nano used in this project does not provide a hardware H.264 encoder (NVENC), so H.264 encoding is handled by `x264enc` on the CPU. Attempted NVENC pipelines failed and fell back to software encoding.

To keep encoding practical:

- Stream bitrate is set to a moderate value (10 Mbps) and `x264enc` is configured to use multiple threads so all CPU cores can participate in encoding.

This change, combined with the thread decoupling and drawing optimizations, allow the pipeline to maintain 24 FPS streaming under realistic Jetson resource constraints.

## Final Pipeline State and Behavior

### Thread Model

The final pipeline uses three main threads on the Jetson:

- Capture loop: Reads frames from the camera via GStreamer and writes the latest `FramePacket` into `_raw_latest`, dropping older frames if needed.
- AI loop: Consumes frames from `_next_frame_for_processing`, runs YOLO with ByteTrack tracking (`track` with fallback to `predict`), extracts detection metadata, and updates shared detection state without blocking the encoder.
- Encoder loop: Independently pulls the newest raw frame and latest detection set, draws overlays, and pushes frames into the output GStreamer pipeline feeding MediaMTX.

Synchronization is handled via a small set of locks (`_raw_lock`, `_detections_lock`, `_tracking_lock`, `_encode_lock`) and monotonic sequence numbers so that each thread operates on the freshest available data without waiting on others.

### Tracking Mode and Re-acquisition

The final pipeline runs YOLO in `track` mode using ByteTrack, with:

- A robust fallback to `predict` on error to handle engines that occasionally fail in `track` paths.
- A re‑acquisition strategy that uses object class (base label) and spatial proximity when track IDs change, preserving tracking continuity after ID swaps or PTZ camera movements.
- Optional smoothing via exponential moving averages on box coordinates to reduce visual jitter inherent in coordinate‑only trackers like ByteTrack.

This configuration balances throughput and stability: tracking ensures PTZ lock‑on behavior for GA8ED’s use cases, while multi‑threading and CPU optimizations maintain high enough FPS for a smooth web experience.

### Integration with FastAPI, MediaMTX, Caddy, and Tailscale

The dashboard uses the encoded stream served via MediaMTX (accessed through Caddy on EC2) while receiving inference metrics and PTZ commands through FastAPI endpoints. Golden‑run scripts and systemd services ensure that Uvicorn, MediaMTX, the GStreamer bridge, and related processes all restart automatically on boot, making the pipeline resilient for field deployments.

Tailscale remains installed but is reserved for SSH and emergency debugging, rather than for normal user access to the dashboard or video. This keeps the production pipeline aligned with standard web expectations (HTTPS, DNS, reverse proxy) while retaining a secure back door for developers when LTE or routing issues appear.

Overall, the code and architecture changes described here moved the system from a monolithic, CPU‑bound YOLO loop at roughly 12 FPS to a multi‑threaded pipeline where GPU inference with ByteTrack, tracking, and rendering are balanced, and where web and network components (MediaMTX, Caddy, and optional Tailscale for diagnostics) work together to deliver stable remote access over LTE.
