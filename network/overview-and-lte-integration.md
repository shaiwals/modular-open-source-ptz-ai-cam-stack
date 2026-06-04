# Modular Open-Source PTZ AI Camera Network Stack and Telit LE910C4-NF Integration

## Overview

This document consolidates the design and implementation work for a modular, open-source PTZ AI camera platform with interchangeable power/network backplates and an LTE-based network stack built around the Telit LE910C4-NF on Jetson Orin Nano (and future COMs such as Verdin i.MX8M Plus). It combines the high-level project specification, the mechanical/electrical backplate architecture, and the full Telit LTE bring-up and troubleshooting history into a single setup guide aimed at reproducible open-source release.

The focus is on: (1) the standardized backplate interface and how it maps from dev kit to custom carrier, (2) the LTE/Wi‑Fi/PoE backplate designs and module choices, and (3) the end-to-end Telit LE910C4-NF configuration on Ubuntu/JetPack, including all major issues encountered and how they were resolved.

## Project Goals and Scope

The project goal is to create a modular, open-source PTZ AI camera platform with on-device intelligence, pan-tilt-zoom motion, and swappable power/network backplates (PoE, Wi‑Fi, LTE, USB‑C, battery, solar), suitable for neighborhood monitoring and research deployments. The system is intended to be reproducible from open CAD, firmware, and software under permissive licenses (MIT for code, CERN OHL for hardware), so documentation must fully capture the wiring, configuration, and boot-time behavior required to bring up the network stack.

Scope for the network and LTE-related work includes:
- Defining a common backplate connector that carries power, USB, and optionally Ethernet/PCIe, abstracting away the specific radio or power modules.
- Implementing a middle "core backplane adapter" PCB that plugs into existing Jetson Orin Nano dev kit connectors and exposes that common interface.
- Selecting and integrating LTE modules (initially Quectel EC25-AF, later Telit LE910C4-NF) via USB, and validating SIM options and data plans for video streaming.
- Bringing up the Telit LE910C4-NF on Ubuntu/JetPack, including driver binding, ModemManager/NetworkManager integration, and performance tuning.

## System Architecture

### High-Level PTZ AI Camera Platform

The overall system consists of a core compute/vision module (initially Jetson Orin Nano, with benchmarking against Raspberry Pi and ESP32-S3) mounted in a PTZ-enabled enclosure with a standardized opening for pluggable power/network backplates. The PTZ base provides 2‑axis motion with servo/stepper drivers and IMU feedback, while the network stack delivers IP connectivity via PoE, Wi‑Fi 6, or 4G LTE, depending on which backplate is installed.

Within the project tracks, the Networking track is responsible for defining the backplate interface, implementing PoE, Wi‑Fi, and LTE modules, and ensuring that module swaps can be performed within roughly 30 seconds without rewiring the rest of the system.

### Backplates

On the Jetson Orin Nano platform, designing a backplate was a low priorty and didn't make much progress during this 6 month project. The Jetson comes with an RJ45 port and multiple USB ports that made designing an extra interface on an external PCB redundant, so it wasn't completed.

### Backplate Types (Future reference)

Backplates are designed around the common connector so that the compute module sees only standardized buses regardless of the specific radio or power hardware.

- PoE backplate:
  - Uses Ethernet TX/RX pairs and power rails to implement IEEE 802.3af PoE PD circuitry and a DC‑DC converter to feed 12 V back into the core.
  - Hosts the external RJ45 jack and any surge/ESD protection.

- LTE backplate:
  - Uses the USB D+/D− pair and 12 V rails to host a USB-based LTE modem such as Quectel EC25‑AF or Telit LE910C4‑NF on a mini PCIe to USB sled, along with SIM slot and antenna connectors.
  - Presents itself to the Jetson purely as a USB device; all LTE complexity stays on that backplate.

- Wi‑Fi backplate:
  - Either uses the Jetson’s built-in M.2 E‑key slot for Wi‑Fi (recommended for the dev kit), or hosts a USB/PCIe Wi‑Fi module connected via USB/PCIe pins from the common connector on custom carriers.

## Rationale for USB-Centric Design

### Why USB for LTE and Wi‑Fi

The design standardizes on USB 2.0 as the primary bus for LTE and optional Wi‑Fi on the dev kit because USB provides a widely supported, easy-to-route interface for both radio types. Many LTE modules and some Wi‑Fi modules expose a USB interface that appears as serial ports or network devices under Linux, avoiding PCIe bring-up and high-speed layout constraints.

USB 2.0 requires only four signals (D+, D−, 5 V, GND), which simplifies connector design and PCB routing over short distances compared to M.2/PCIe with multiple high-speed differential pairs and sideband pins. Since the Orin Nano dev kit already exposes USB host ports but not raw PCIe lanes on headers, USB is the most practical common denominator in a student prototype context.

### PCIe and Other Buses

The Orin Nano dev kit does provide PCIe lanes via its M.2 slots (Key E and Key M), which can be used directly for Wi‑Fi cards on the E‑key slot and NVMe SSDs on the M‑key slots. However, breaking those lanes out through additional adapters into a common connector for backplates adds mechanical and signal integrity complexity, and LTE modules often require B‑key or mini PCIe form factors that do not match the E‑key slot natively.

For a future custom carrier, the recommended approach is to keep USB on the common connector and add PCIe x1 and/or native RGMII/SGMII Ethernet lanes, letting higher-end Wi‑Fi or LTE modules use PCIe directly while preserving USB for simpler modules.

## LTE Module Selection and SIM Options

### Quectel EC25-AF (Background)

Earlier design iterations considered the Quectel EC25‑AF mini PCIe LTE Cat 4 modem, which uses USB 2.0 as its host interface over the mini PCIe edge connector. This makes it an excellent fit for the USB-based backplate architecture: a mini PCIe socket or a dedicated mini PCIe‑to‑USB adapter sled on the LTE backplate can present the modem as a USB device to the Jetson.

The EC25‑AF supports North American LTE bands and data rates up to approximately 150 Mbps downlink and 50 Mbps uplink, which is sufficient for 720p H.264 streaming given typical bitrates of a few Mbps per stream. It is used as a reference point for selecting comparable modules.

### Telit LE910C4-NF

Due to its better availability, the project uses the Telit LE910C4‑NF LTE Cat 4 module in a mini PCIe form factor accessible over USB 2.0. The NF variant is a North American-focused model with 3GPP Release 10 compliance and LTE Cat 4 performance, and is documented to work with major US carriers when configured with the appropriate APN and band mask.

On Linux, the LE910C4‑NF exposes one or more USB serial ports (e.g., /dev/ttyUSB2, /dev/ttyUSB3) and can operate in different USB compositions, including pure serial and ECM/NCM network modes, configurable via AT commands. This flexibility allows tuning for performance and CPU overhead once basic connectivity is established.

### SIM and Data Plan Choices

The project explores multiple SIM options for LTE connectivity:
- Google Fi data‑only SIMs, which provide LTE access on T‑Mobile/US Cellular in the US and are compatible with the LE910C4 series when the APN is set to `h2g2`.
- IoT-focused data SIMs (Sixfab, Simbase, Hologram, 1NCE, etc.), which offer pooled or pay‑per‑MB plans suitable for sensor traffic but can become expensive for continuous HD video streaming.
- Standard prepaid consumer data SIMs from major carriers (AT&T, T‑Mobile, Verizon) or MVNOs (Mint, Boost, SpeedTalk), which are more cost-effective for higher data usage such as 720p video streaming in a prototype.

For this project, a Google Fi data‑only SIM is used during Telit bring-up, with APN `h2g2` and band configuration tuned for T‑Mobile’s LTE bands in Seattle.

## Telit LE910C4-NF Hardware Integration

### Physical Connection on Jetson Orin Nano

The Telit LE910C4‑NF module is installed on a mini PCIe to USB adapter sled with a SIM slot, which is then plugged directly into the Jetson's USB port. The sled breaks out the mini PCIe edge connector to USB D+/D−, 5 V, and ground, effectively turning the LTE card into a USB modem from the Jetson’s perspective.

### USB Enumeration and Driver Binding

Once the Jetson and Telit module are powered, `lsusb` is used to verify that a Telit device with vendor ID 0x1bc7 appears, confirming basic USB enumeration. On some Jetson kernels, the generic `option` or `usbserial` drivers do not auto-bind to the Telit VID/PID, so manual binding via `modprobe` and `echo` into `/sys/bus/usb-serial/drivers/option/newid` is used as needed, along with udev rules for persistence.

There are Jetson-specific quirks when loading `usbserialvendor`, which may not exist as a separate module on L4T 5.10 kernels; instead, `usbserial` and `option` modules are loaded and the Telit VID/PID is added via `modprobe usbserial vendor=0x1bc7 product=0xXXXX` where `XXXX` is the product ID from `lsusb`.

## Linux Software Stack on JetPack

### Base Services: ModemManager and NetworkManager

On Ubuntu/JetPack, ModemManager and NetworkManager are used as the primary control stack for LTE modems. The installation process ensures both services are present without upgrading the OS:
- `apt update` or a cautious `apt install --no-install-recommends modemmanager network-manager` using existing package indices, avoiding distribution upgrades.
- Enabling and starting services via `systemctl` or `service`, with fallback strategies when `systemctl` is unavailable or systemd is not visible in minimal environments.

### Detecting the Modem

After services are running and USB drivers are bound, ModemManager should list the Telit modem using `mmcli -L`, which yields an entry like `Modem /org/freedesktop/ModemManager1/Modem/0`. If `mmcli -L` shows no modems, troubleshooting focuses on:
- Verifying Telit presence via `lsusb | grep 1bc7`.
- Checking `/dev/ttyUSB*` or `/dev/ttyACM*` devices and ensuring no stale `pppd` processes are holding ports open.
- Confirming that udev rules do not accidentally blacklist the modem.

Once detected, `mmcli -m 0` is used to inspect the modem state and confirm primary AT and data ports.

## APN and Registration Configuration

### AT Command Access

The primary AT command port on the Telit LE910C4‑NF is typically `/dev/ttyUSB3`, with secondary or diagnostic ports on other `/dev/ttyUSBx` devices. Terminal tools such as `minicom` or `screen` are used to send AT commands for diagnosis and configuration.

Common initial commands include:
- `AT` (basic connectivity check) and `AT+CMEE=2` (enable verbose error reporting).
- `AT+CSQ` for RSSI/BER and `AT+CEREG?` / `AT+CREG?` for LTE/GSM registration status.
- `AT+COPS?` to check operator selection.

### Network Registration and Band Configuration

In early setup, a situation was observed with excellent signal (e.g., `+CSQ: 31,0` indicating very strong RSSI) but `+CME ERROR: 100` errors indicating "no network service". Debugging steps included:
- Checking registration via `AT+CREG?` and `AT+CEREG?`, expecting status `1` or `5` for registered (home/roaming).
- Verifying SIM status with `AT+CPIN?` and `AT+CIMI`.
- Confirming that the modem’s band configuration matched Google Fi / T‑Mobile bands in Seattle via `AT#BND?` and `AT#BND=`.

The command `AT#BNDSEL` produced errors on the NF variant; the correct command for this module is `AT#BND`, with a mask such as `#BND: ,10,80800000000381A` representing an appropriate LTE band set. Once a valid band mask was applied and the modem rebooted, `+CREG`/`+CEREG` indicated successful network registration.

### APN Setup

With registration established, the APN is configured for Google Fi using:
- `AT+CGDCONT=1,"IP","h2g2"` to set the PDP context.
- `AT+CGACT=1,1` to activate the context and `AT+CGPADDR=1` to check the assigned IP.

In ModemManager, the same APN is provided via:
- `mmcli -m 0 --simple-connect="apn=h2g2"` for straightforward connection.

- Creating a persistent NetworkManager GSM connection with `nmcli c add type gsm ... apn h2g2` if NM-based management is desired.

## Connectivity Methods and Boot Persistence

### Direct ModemManager Control (Recommended)

After successful registration and APN configuration, the primary connectivity path uses ModemManager:
- `mmcli -m 0 --enable` to ensure the modem is powered and enabled.
- `mmcli -m 0 --simple-connect="apn=h2g2,ip-type=ipv4"` to establish a bearer and bring up a WWAN interface (e.g., `wwan0` or `usb0`).

This approach leverages ModemManager’s support for Telit modems and allows monitoring of signal and bearer status via `mmcli -m 0 --signal-get` and `mmcli -b`.

To ensure automatic reconnection at boot, a systemd unit is created that runs `mmcli --simple-connect` after ModemManager starts, setting the modem index appropriately and keeping the bearer connected for headless prototypes.

### NetworkManager GSM Connection

NetworkManager can manage the LTE connection via a GSM profile:
- `nmcli c add type gsm ifname "*" con-name Fi apn h2g2` to create a connection profile.
- `nmcli c up Fi` to bring the connection up and establish default routes.

Autoconnect is enabled with `nmcli c modify Fi connection.autoconnect yes` so that NM restores the LTE connection on reboot without manual intervention.

In Jetson environments, NM and ModemManager interactions can sometimes interfere with each other or with Wi‑Fi; hence, a hybrid approach is used where ModemManager owns the LTE modem while NetworkManager primarily manages Wi‑Fi, with udev rules and masks preventing NM from misconfiguring the LTE ports.

### PPP Fallback

As an alternative to ModemManager/NM, PPP-based connectivity is explored for minimal or custom environments:
- Creating a `/etc/ppp/peers/telit` file pointing to the primary AT port (e.g., `/dev/ttyUSB3`) with options such as `115200`, `noauth`, `ipparam telit`, and `usepeerdns`.
- Defining a chat script that issues `ATZ`, sets `AT+CGDCONT=1,"IP","h2g2"`, and dials `*99#` or `ATD*99#` as appropriate.
- Adding a `ppp0` stanza to `/etc/network/interfaces` to bring PPP up at boot.

This method achieved connectivity but exhibited higher jitter and packet loss under load compared to optimized ECM/NCM/WWAN modes.

## Performance Issues and Optimization

### Initial Performance Problems

Initial speed tests with the Telit module showed high latency (over 1200 ms), approximately 86% packet loss, and severe jitter, despite excellent reported signal strength (e.g., `RSRP ≈ -64 dBm`, `RSRQ ≈ -3 dB`). This pointed to link-layer or configuration issues rather than coverage problems.

Contributing factors considered included:
- Single-antenna operation (no MIMO), limiting throughput and robustness.
- Suboptimal LTE band selection with congested or weak bands.
- PPP overhead and serial link inefficiencies.
- Jetson bridge configuration (`l4tbr0`) interfering with raw IP interfaces.  

### Moving from PPP to ECM/NCM

PPP over a serial port introduced substantial overhead; repeated tests indicated that moving to network-interface modes (ECM/NCM) significantly improves performance and stability. The Telit module’s USB composition is configured via `AT#USBCFG`:
- `AT#USBCFG?` to query the current mode.
- `AT#USBCFG=1` to enable ECM mode for network-over-USB.
- `AT#REBOOT` to apply changes.

After reboot, the module exposes a `usb0` Ethernet interface which can be brought up via `ip link set usb0 up` and `dhclient usb0` or equivalent DHCP client, resulting in dramatically reduced packet loss and improved throughput.

### Jetson Network Bridging and DHCP Issues

On JetPack, NVIDIA’s default `l4tbr0` bridge and network configuration can interfere with USB network interfaces. Symptoms included `usb0` showing "UNKNOWN" state, or `dhclient` failing with `RTNETLINK answers: file exists` or no lease.

Resolutions included:
- Removing `usb0` from `l4tbr0` using `brctl delif l4tbr0 usb0` and bringing it up as a standalone interface.
- Flushing stale IP and route state via `ip addr flush dev usb0` and then re-running `dhclient -v usb0`.
- In cases of persistent DHCP issues, using `busybox udhcpc -i usb0` or temporarily assigning a static IP within the modem’s private subnet for testing.

Once these changes were applied, speed tests showed stable throughput in the several Mbps range and near-zero packet loss, with further gains expected from dual-antenna and band-optimization strategies.

### Antenna and Band Tuning

The antenna was initially set up incorrectly, with the ports swapped. Once the ports were properly assigned, the network speeds dramatically improved, which indicated that the antenna orentation was the most important factor.

## Error Conditions and Resolutions

This section summarizes notable issues encountered and how they were resolved, to serve as a troubleshooting reference.

### CME Error 100 with Strong Signal

- Symptom: `AT+CSQ` reports strong signal (`31,0`), but follow-on commands (e.g., `AT+NWMODE`, `AT+USBPORT`) return `+CME ERROR: 100` or similar "unknown" errors.
- Root cause: Incorrect or unsupported AT commands for the LE910C4‑NF variant, or modem not yet registered to network despite RF visibility.
- Fix: Use correct Telit commands (`AT#BND` instead of `AT#BNDSEL`, avoid unsupported `AT#NWMODE`/`AT#USBPORT` where firmware does not support them), verify registration with `AT+CREG?`/`AT+CEREG?`, and reset radio via `AT+CFUN=1,1` when needed.

### Missing `/sys/bus/usb-serial/drivers/option1/newid`

- Symptom: Attempts to echo VID/PID into `/sys/bus/usb-serial/drivers/option1/newid` fail because the directory does not exist.
- Root cause: `option` driver not loaded, or Jetson kernel naming drivers differently (e.g., `option` without `option1`).
- Fix: Load appropriate modules via `modprobe usbserial` and `modprobe option`, then use the actual driver directory that appears under `/sys/bus/usb-serial/drivers/` (often `option` rather than `option1`).

### `usbserialvendor` Module Not Found

- Symptom: `modprobe usbserialvendor` fails with "module not found" on Jetson kernel 5.10.120‑tegra.
- Root cause: `usbserialvendor` is not a separate module in this kernel; vendor parameters must be provided to `usbserial` instead.
- Fix: Use `modprobe usbserial vendor=0x1bc7 product=0xXXXX` or rely on `option` with `newid` entry, and install `linux-modules-extra` packages if necessary.

### Modem Missing from `mmcli -L`

- Symptom: After reconfiguration or disconnect, `mmcli -L` reports "no modems found" even though the Telit device appears under `lsusb`.
- Root cause: ModemManager crashed or is stuck in an inconsistent state; udev rules may have blacklisted the device, or the Telit has changed USB composition.
- Fix: Restart ModemManager (`systemctl restart ModemManager`), unplug/replug the modem, and ensure udev rules that ignore Telit during PPP experiments are reverted when switching back to ModemManager.

### `usb0` Link Up but No IP

- Symptom: `ip link show usb0` indicates `UP` with "carrier detected", but no IPv4 address is assigned and pings fail.
- Root cause: DHCP client conflicts, `l4tbr0` bridge configuration, or unsupported raw-IP behavior in standard DHCP clients.
- Fix: Remove `usb0` from bridges, flush addresses, rerun `dhclient -v usb0` or use alternative clients; in some cases, manually assign an IP in the modem’s private range for connectivity testing.

### Wi‑Fi Disruption When Restarting NetworkManager

- Symptom: Restarting NetworkManager as part of LTE testing causes Wi‑Fi connectivity to drop and not automatically recover.
- Root cause: NetworkManager manages both Wi‑Fi and LTE; restarts disrupt active Wi‑Fi connections and can leave interfaces in a down state.
- Fix: After restarts, explicitly re-enable Wi‑Fi via `nmcli radio wifi on` and reconnect using `nmcli c up <wifi-connection>`, or isolate LTE from NM by using ModemManager-only control and leaving NM to manage Wi‑Fi exclusively.

## Portability to Verdin i.MX8M Plus and Other COMs

Although initial work is on Jetson Orin Nano, the LTE stack is designed to be portable to other ARM-based COMs such as Toradex Verdin i.MX8M Plus on the Dahlia carrier, provided that the LTE module continues to enumerate as a USB modem. Toradex’s BSPs support ModemManager/NetworkManager and include on-module Wi‑Fi/Bluetooth (on WB variants), simplifying the network stack to LTE via USB and Wi‑Fi via the SoM, with Tailscale running on Linux/arm64.

Portability checks focus on:
- Verifying that Telit LE910C4‑NF USB VID/PID and composition are supported by the new kernel or ModemManager version.
- Ensuring that no Jetson-specific udev rules or driver quirks are assumed.
- Confirming that Tailscale operates correctly on the target OS and architecture.

Because the design standardizes on USB as the host interface, moving from Jetson to Verdin primarily requires re-testing driver support and adjusting any systemd or NM scripts to match the new device naming and service configuration.

## Standards, Certifications, and Licensing

### Communication and Interface Standards

The LTE subsystem is built on the 3GPP LTE standard, with the Telit LE910C4‑NF implementing 3GPP Release 10 LTE Cat 4 capabilities. Wi‑Fi modules are expected to comply with IEEE 802.11 standards (e.g., 802.11n/ac/ax), and Ethernet links and PoE modules use IEEE 802.3 and 802.3af/at specifications for data and power over twisted pair.

The Jetson Orin Nano platform itself adheres to USB 3.2, HDMI 2.0, MIPI CSI‑2, and IEEE 802.3 Ethernet interface standards, and carries regulatory certifications such as FCC, CE, and KC, which form part of the system-level compliance story when combined with certified LTE and Wi‑Fi modules.

### Telit LE910C4-NF Certifications

The Telit LE910C4‑NF module has 3GPP LTE compliance and is listed with North American carrier approvals including FCC/ISED and PTCRB, with additional regional certifications depending on variant and market. These certifications enable its use on major US carriers as long as network operators approve the specific hardware configuration.

### Open-Source Licensing

The project intends to release firmware and software under the MIT license and hardware designs (PCBs, mechanical CAD) under CERN OHL, enabling broad academic and industrial reuse while requiring attribution and license preservation for derivative hardware designs. All setup and troubleshooting documentation, including this network stack guide, is written to support such an open-source release by providing complete configuration steps and rationale.

## Summary of Key Lessons Learned

Key lessons from the LTE/network portion of the project include:
- Standardizing on a USB-based backplate interface greatly simplifies LTE and Wi‑Fi integration on dev kits that expose USB but not raw PCIe headers, while remaining portable to future carriers that can add PCIe lanes later.
- The Telit LE910C4‑NF can be made to work reliably on Jetson Orin Nano with a Google Fi data‑only SIM, but requires careful attention to USB driver binding, APN configuration, LTE band masks, and Jetson-specific network bridge behavior.
- PPP-based connectivity is functional but suboptimal; enabling ECM/NCM modes and treating the modem as a USB Ethernet device provides significantly better performance and stability for video streaming workloads.
- Many hard-to-diagnose issues (e.g., strong signal with no service, link up with no IP, Wi‑Fi loss when restarting NM) were resolved by systematically separating responsibilities between ModemManager, NetworkManager, and low-level drivers, and by keeping a clear mental model of how the Jetson’s default network configuration interacts with USB WWAN interfaces.

Capturing these details in open documentation should allow future contributors to reproduce the network stack, adapt it to alternative compute modules, and focus on higher-level AI and PTZ features without re-solving low-level LTE integration problems.
