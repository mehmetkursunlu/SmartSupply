# SmartSupply
SmartSupply is a digital battery powered lab powersupply. It is Arduino compatible and can be controlled via PC over USB.


## Specifications:
 * 0 - 1A,  steps of 1 mA  (10 bit DAC)
 * 0 - 20V, steps of 20 mV (10 bit DAC) (true 0V operation)
 * Voltage measurement: 20 mV resolution (10 bit ADC)
 * Current measurement: 
      -     < 40mA:  10uA resolution (ina219)
 			< 80mA:  20uA resolution (ina219)
 			< 160mA: 40uA resolution (ina219)
			< 320mA: 80uA resolution (ina219)
			> 320mA: 1mA  resolution (10 bit ADC)

## Features: 
 * Constant voltage and constant current modes
 * Uses a low noise linear regulator, preceded by a tracking preregulator to minimize power dissipation
 * Aluminium case end panel used as heatsink 
 * Use of handsolderable components to keep the project accessible 
 * Powered by ATMEGA328P, programmed with Arduino IDE
 * PC communication via Java application over micro usb
 * Powered by 2 protected 18650 Lithium Ion cells
 * 18 mm spaced banana plugs for compatibility with BNC adapters

![SmartSupply](https://github.com/ThomasVDD/SmartSupply/blob/master/Pictures/Front.jpg)
![SmartSupply](https://github.com/ThomasVDD/SmartSupply/blob/master/Pictures/Inside.jpg)
![SmartSupply](https://github.com/ThomasVDD/SmartSupply/blob/master/Pictures/Back.jpg)
![SmartSupply](https://github.com/ThomasVDD/SmartSupply/blob/master/Pictures/PC.jpg)

## Theory of operation: 

To understand the operation of the circuit, we will have to look at the schematic. I divided it into functional blocks, such that it is easier to understand; I will thus also explain the operation step by step. 

### Main block
The operation is based around the LT3080 chip: it's a linear voltage regulator, which can step down voltages, based on a control signal. This control signal will be generated by a microcontroller; how this is done, will be explained in detail later. 

#### Voltage setting
The circuitry around the LT3080 generates the appropriate control signals. First, we will take a look at how the voltage is set. 
The voltage setting from the microcontroller is a PWM signal (PWM_Vset), which is filtered by a lowpass filter (C9 & R26). This produces an analog voltage - between 0 and 5 V - proportional to the wanted output voltage. Since our output range is 0 - 20 V, we will have to amplify this signal with a factor of 4. This is done by the non inverting opamp configuration of U3C. The gain to the set pin is determined by R23//R24//R25 and R34. These resistors are 0.1% tolerant, to minimize errors. R39 and R36 don't matter here, as they are part of the feedback loop.


#### Current setting
This set pin can also be used for the second setting: current mode. We want to measure the current draw, and turn the output off when this exceeds the wanted current. Therefore, we start again by a PWM signal (PWM_Iset), generated by the microcontroller, which is now lowpass filtered and attenuated to go from a 0 - 5 V range to a 0 - 2 V range. This voltage is now compared to the voltage drop accross the current sense resistor (ADC_Iout, see below) by the comparator configuration of opamp U3D. If the current is too high, this will turn on an led, and also pull the set line of the LT3080 to ground (via Q2), thus turning off the output.

The measurement of the current, and the generation of the signal ADC_Iout is done as follows. The output current flows through resistors R7 - R16. These total 1 ohm; the reason for not using 1R in the first place is twofold: 1 resistor would need to have a higher power rating (it needs to dissipate at least 1 W), and by using 10 1% resistors in parallel, we get a higher precision than with a single 1 % resistor. A good video about why this works can be found here: https://www.youtube.com/watch?v=1WAhTdWErrU&t=1s
When current flows through these resistors, it creates a voltage drop, which we can measure, and it is placed before the LT3080, since the voltage drop accros it should not influence the output voltage. 
The voltage drop is measured with a differential amplifier (U3B) with a gain of 2. This results in a voltage range of 0 - 2 V (more on that later), hence the voltage divider at the PWM signal of the current. The buffer (U3A) is there to make sure that the current flowing into resistors R21, R32 and R33 is not going through the current sense resistor, which would influence it's reading. 
Also note that this should be a rail-to-rail opamp, because the input voltage at the positive input equals the supply voltage. 

The non inverting amplifier is only for the course measurement though, for very precise measurements, we have te INA219 chip on board. This chip allows us to measure very small currents, and is addressed via I2C. 

#### Additional things
At the output of the LT3080, we have some more stuff. First of all, there is a current sink (LM334). This draws a constant current of 677 uA (set by resistor R41), to stabilize the LT3080. It is however not connected to ground, but to VEE, a negative voltage. This is needed to allow the LT3080 to operate down to 0 V. When connected to ground, the lowest voltage would be about 0.7 V. This seems low enough, but keep in mind that this prevents us from turning the powersupply completely off. 
The zener diode D3 is used to clamp the output voltage if it goes above 22 V, and the resistor divider drops the output voltage range from 0 - 20 V to 0 - 2 V (ADC_Vout). 
Unfortunately, these circuits are at the output off the LT3080, which means their current will contribute to the output current we want to measure. Fortunately, these currents are constant if the voltage stays constant; so we can calibrate the current when the load is disconnected first. 

### Charge pump
The negative voltage that we mentioned before is generated by a curious little circuit: the charge pump. For it's operation, I would refer to here: https://www.youtube.com/watch?v=LtoPHevexTM&t=2s It is fed by a 50% PWM of the microcontroller (PWM)

### Boost Converter
Let's now take a look at the input voltage of our main block: Vboost. We see that it is 8 - 24V, but wait, 2 lithium cells in series gives a maximum of 8.4 V? Indeed, and that's why we need to boost the voltage, with a so called boost converter. We could always boost the voltage to 24 V, no matter what output we want; however, this would waste a lot of power in the LT3080 and things would get toasty hot! So instead of doing that, we will boost the voltage to a bit more than the output voltage. About 2.5 V higher is appropriate, to account for the voltage drop in the current sense resistor and the dropout voltage of the LT3080. 
The voltage is set by resistors on the output signal of the boost converter. To change this voltage on the fly, we use a digital potentiometer, the MCP41010, which is controlled via SPI. 

### Battery Charging
This leads us to the real input voltage: the batteries! Since we use protected cells, we simply need to put them in series and we're done! It is important to use protected cells here, to avoid overcurrent or overdischarge, and thus damaging, of the cells. 
Again, we use a voltage divider for measuring the battery voltage, and dropping it down to a usable range.
Now on to the interesting part: the charging circuitry. We use the BQ2057WSN chip for this purpose: in combination with the TIP32CG, it basically forms a linear powersupply itself. This chip charges the cells via an appropriate CV CC trajectory. Since my batteries don't have a temperature probe, this input should be tied to half the battery voltage. This concludes the voltage regulation part of the powersupply. 

### 5V regulator
The 5 V supply voltage of the arduino is made with this simple voltage regulator. It is not the most precise 5 V output however, but this will be solved below. 

### 2.048 V voltage reference
This little chip provides a very accurate 2.048 V voltage reference. This is used as a reference for the analog signals ADC_Vout, ADC_Iout, ADC_Vbatt. That's why we needed voltage dividers to bring these signals down to 2 V. 

### Microcontroller
The brain of this project is the ATMEGA328P, this is the same chip that is used in the Arduino Uno. We already went over most control signals, but there are some interesting additions nonetheless. The rotary encoders are connected to the 2 only external interrupt pins of the arduino: PD2 and PD3. This is needed for a reliable software implementation. The switches underneath use an internal pullup resistor. 
Then there is this strange voltage divider on the chip select line of the potentiometer (Pot). A voltage divider on an output, what's that good for; you might say. As mentioned before, the 5 V supply is not teribbly accurate. It would thus be good to measure this accurately, and adjust the duty cycle of the PWM signal accordingly. But since I had no more free inputs, I had to make a pin pull double duty. When the powersupply boots, this pin is first set as an input: it measures the supply rail and calibrates itself. Next, it is set as an output and it can drive the chip select line. 

### Display Driver
For the display, I wanted a commonly available - and cheap - hitachi lcd screen. They are driven by 6 pins, but since I had no pins left, I needed another solution. A shift register to the rescue! The 74HC595 allows me to use the SPI line to control the display, thus only needing 1 additional chip select line. 

### FTDI
The last part of this powersupply is the connection with the cruel, outside world. For this, we need to convert the serial signals into USB signals. This is done by an FTDI chip, which is connected to a micro usb port for easy connection.

And that's all there is to it! 

## Licensing and acknowledgements

![License](https://github.com/ThomasVDD/SmartSupply/blob/master/Pictures/License.PNG)

This project would never have been possible without the help of some people, so a shootout seems appropriate.
 * This project is based on EEVBLOG's uSupply project and his Rev C schematic. So a special thanks to David L. Jones for releasing his schematics under an open source license and sharing all his knowledge.
 * A huge thanks to Johan Pattyn for producing the prototypes of this project. 
 * Also Cedric Busschots and Hans Ingelberts deserve credit for the help with troubleshooting.
