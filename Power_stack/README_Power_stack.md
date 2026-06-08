# ***Power Stack***
## **Overview**
DO NOT MANUFACTURE BEFORE REVISION. THIS VERSION HAS CRITICAL ERRORS THAT NEED TO BE FIXED. 
The goal of this is to supply power the the PTZ camera system via PoE (power over etherent), solar, and a battery. 
The current board is able to supply power via battery only. 

## **Core Design** 
### **key datasheets**
AG5810: https://silvertel.com/images/datasheets/Ag5810-datasheet-IEEE802_3bt-Power-over-Ethernet-4-pair-PD.pdf
BQ25798: https://www.ti.com/lit/ds/symlink/bq25798.pdf?ts=1780730528262&ref_url=https%253A%252F%252Fwww.ti.com%252Fproduct%252FBQ25798
IRLB4783: https://www.infineon.com/assets/row/public/documents/24/49/infineon-irlb8743-datasheet-en.pdf?fileId=5546d462533600a4015356605d6b2593
IHTH0750IZEB1R0M5A: https://4donline.ihs.com/images/VipMasterIC/IC/VISH/VISH-S-A0011391953/VISH-S-A0011391953-1.pdf?hkey=CECEF36DEECDED6468708AAF2E19C0C6
### **BQ25798**
The design of the power stack centers around the BQ25798 
The IC is designed to handle multiple input sources including a solar source and be actively managed by a host via I2C. 
The first function of the IC is to monitor and charge the battery. if there is not a source to charge the battery from, the IC will connect the battery to the sys or output rail allowing the battery provide power to the load. 
If there a source or multiple sources present the IC will choose between them and will connect the source to the battery allowing the battery to charge. The IC will also allow the new source to simultaneously serve the load. The BQ25798 has its own source validation functions that it performs automatically. 
I chose to use a reference design provided in the data sheet on page  130. The differences are as follows
1. I needed a 10.5k ohm resistor at the prog pin since we wanted to operate for a 3s 1.5MHz configuration
2. the resistor connected to ILIM_HIZ are both 100k ohm since I wanted to limit imput current less
3. I implemented a snubber with a 10 ohm resistor and a 1nF capacitor in parallel with the inductor (more on this in section X)
4. the 2.2k ohm resistor connected to the LED, REGN and STAT is a 5.6k resistor to adjust for the LED I used
5. The 294 ohm resistors are implemented as 330 ohm resistors because I did not have 294 ohm resistors. 
6. ALL optional items are implemented
for details on how to select an inductor for the chip see the datasheet pages 133-134
I have not done anything with the I2C interface
This configuraiton is best for solar cells with an additional adapter which is what we were aiming to do. 
### **PoE path design**
PoE must first negotiate power before formally delivering it. To handle this neogtiation, I chose the silvertel AG5810. 
We never got the opportunity to formally implement a proper way to split the power lines and the data lines for ethernet. Therefore, the quickest way to implement was to use diodes to tie the power lines together. This procedure is explained in the datasheet of the silvertel AG5810. This has the consequence of corrupting the data that may have been on those lines. This will likely need to be changed in the future to accomodate data over ether net as well as power. 
I wanted the adjust the output voltage of the AG5810 up, so I used a 8200 ohm resistor on pin 13 to tell the module to up the output voltage to 15V
I used the Parallel output configuration shown on page 8 of the AG5810 datasheet. 
PoE is very sensitive to inrush current. I found that my design would frequently fail to provide power via PoE becuase the current requested was causing the voltage to collapse. To help mitigate this issue, I added an additional power stage. This addiitonal power stage is just a boost converter. The inductor on the boost converter was able to limit current and allowed me to add more bulk capacitance to help alleviate inrush current causing voltage collapse. A similar function can likely be accomplished with a series inductor in future iterations. 
At the output of the boost converter, the output rail is connected to the BQ25798 through power MOSFETs. 
### **Solar path design**
This is just a copy from the TI data sheet. Have not done any work validating it. 
### **System Output Rail design**
We needed to accomadate a power level for both the motors and the camera system. To accomplish this, I split the BQ25798 output into 2 seperate parallel converter paths. 1 buck converter for the motors and 1 boost converter for the camera system. 
### **PoE and battery initialization and function**
1. battery is plugged it
2. plug in the ehternet cable. 
3. the AG5810 negotiates power over ethernet
4. the AG5810 reads pin 13 for output voltage. 
5. the voltage propagates through to the Power MOSFETs
6. The BQ25798 checks the voltage of PoE using the VAC1 pin
7. if the BQ25798 senses a good voltage, the ACDRV1 pin rises to 5V driving the the power MOSFETs gate
8. the internal REGN rail on the BQ25798 powers up
9. the converter in the BQ25798 starts switching connecting the PoE source to the rest of the system. 
10. the SYS or output rail is now live and serving a 19V laod for the computer hardware and a 5V source for the motors. 
### Known issues
1. the PCB layout currently presented is not good and needs to be redone. it has resulted in many issues.
2. The decoupling capacitors for the Vbus rail of the BQ25798 are too far from the BQ25798. This results in excessive ringing on the switch nodes which kills the internal mosfets of the bq25798. Once I learned this, I implemented a snubber with 1nF capacitor and a 10ohm resistor and added more capacitance to the Vbus rail to help mitigate the issue. after implementing these, I was able to make the board work for applications no more than 10W total incuding battery charging and load serving. Under higher power conditions I ran into a new issue. the switch node oscillations still looked good, but the REGN voltage relative to ground was spiking uncontrollably and resulted in internal damage to the BQ25798. I do not know the cause. AI thinks it is something called ground bounce, but based on the fact that the REGN voltage killed the BQ25798, I think the AI is wrong. I think it is likely that the layout inductance became too much for the BQ25798 to handle which resulted in critical failure. 
3. the decoupling capacitor for the REGN rail is too far from the chip
Due to an excessive amount of bulk capacitance and sometimes not enough, PoE can sometimes struggle to negotiate power consistently. The half solution I found to this was adding an extra power stage to seperate the AG5810 from the BQ25798. The issue is that PoE voltage is very fragile. if too much current is pulled at once the voltage collapses and the AG5810 fails power negotiation (more capacitors = higher current), but if there is not enough bulk capacitance, the transient load will cause too much current resulting PoE turning off. 
4. The inductor footprint is wrong
5. the 2 ohm hot plug resistor footprint needs to be for a 2W resistor 

### Future items in order of importnace
You will need to validate that the board works after each iteration
1. Redo PCB layout. see BQ25798 datasheet page 140-141
2. add in a properly rated TVS diode at the ethernet and solar cell input
3. Do something about the heat the BQ25798 will produce when providing power. (power converters too)
4. get I2C interface working and configure as necessary. you will need to enable MPPT via I2C for solar panels
5. get real buck and boost converters to replace the ones that are currently on the board. You will need to do inductor selection
6. Resize the inductor used on the BQ25798. The one we have right now is too large
7. adjust the amount of bulk capacitance. right now there is probably too much
8. once all of the above is working well move to implementing a power mux between usb-C power and PoE power using the LTC4416 power mux. 
9. new enclosure design

### Key component selection and what to do with them
1. BQ25798: keep. it suits this project very well
2. AG5810: keep unless there is an issue getting ethernet data
3. IRLB8743: these are power mosfets and have worked very well 
4. 1N4007: these diodes are not the best for the application. they are very lossy
5. Change all capacitors to X7R or X5R
6. purchased from amazon converters: get a real converter and select your own inductor
7. IHTH0750IZEB1R0M5A: make a better smaller choice for an inductor. see pages 133-134 on the BQ25798 datasheet for inductor selection
