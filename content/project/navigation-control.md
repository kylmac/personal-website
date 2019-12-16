---
title: "Navigation Control"
date: 2019-12-15T20:40:45-05:00
draft: false
---

Autonomous Vehicle Navigation Control
===================


The goal of this project was to design and build the electrical and software systems for a motor- controlled vehicle that could successfully navigate two laps of a track within 63 seconds.  A camera is used to detect the car’s position relative to the black line. Steering is controlled by pulse-width modulating the signal delivered to the servo motor through PD control using the camera output as input.

----------


Key Components
-------------
Our vehicle is composed of an electrical subsystem and a software/firmware subsystem. The power, video, and comparator boards belong to the electrical subsystem while the top design and code belong to the software subsystem.

#### Power Board Modification
Every additional component implemented in navigation control is powered using the same power board as used in speed control. However, a designated power source is needed to power the servo motor as it would otherwise draw too much current. This power source is shown below.
![alt text](https://github.com/kylmac/ELE302_Navigation_Control/blob/master/images/PowerBoard.png "yes")

#### Video Board
The primary function of the **video board** is to power the camera and deliver the output of the camera to the PSoC and comparator board respectively. Shown below, the composite video output of the camera is sent to the sync separator and the comparator before finally reaching the PSoC.
![alt text](https://github.com/kylmac/ELE302_Navigation_Control/blob/master/images/Video%20Board.png)
Decoupling capacitors C1 and C2 are used to reduce noise on the power line entering the camera. R1 is an impedance matching resistor. R2, otherwise known as R Set, is chosen at 680kΩ based on the pulse separation of the input signal. There is a range of resistor values that would work, as shown in the **LM1881** datasheet. Using too much resistance would result in the absence of a vertical pulse, while using too little resistance would result in a double vertical pulse. Finally, the raw composite video output is sent to the comparator while the v-sync and h-sync signals are sent to the PSoC.

#### Comparator Board
The **comparator board** is used to distinguish white and black from the camera’s composite video output. The camera’s output is an analog voltage signal that goes low on black and high on white. The comparator digitizes and properly thresholds white and black. As shown below, a potentiometer is used to set the proper threshold voltage for observing white and black.
![alt text](https://github.com/kylmac/ELE302_Navigation_Control/blob/master/images/Comparator.png)

Upon experimentation, we found that the threshold for the camera identifying black correctly was 0.6V . In other words, the camera signal was quantized by mapping all values less than 0.6V to 0 and all values greater than or equal to 0.6V to 1. Finally, because the **LM311** has an open collector output, we use R2, a pull-up resistor, to pull up the output to the desired high value and eliminate any floating values. We chose 20kΩ because a large enough resistor is required to limit the leakage current flowing through the comparator.
#### Top Design: Measuring Car Position
The **top design** for navigation can be broken into two sections:

 1. Measuring the position of the car relative to the black line on the floor
 2.  Sending the appropriate PWM signal to the servo motor


The former, shown below, is accomplished using the three signals sent to the PSoC: the output of the comparator, the **h-sync** signal, and the **v-sync** signal. The output of the comparator will indicate if a black or white pixel is observed. The h-sync and v-sync signals indicate a new line and frame, respectively. The first step is to invert all the signals. The inversion of the h-sync and v-sync is because these signals go low on a new line and frame, but are high otherwise. We found it more useful to have the new lines and frames be represented as rising edges, with the signals going low otherwise, as most components trigger on a rising edge. The output of the comparator is inverted so black was represented by a falling edge and white by a rising edge.
![alt text](https://github.com/kylmac/ELE302_Navigation_Control/blob/master/images/Top%20Design.png)
We then capture one line per frame. A counter is implemented with the h-sync pulse as input and a reset occurring on the rising edge of the v-sync pulse. The output of the counter goes high for the duration of the line of interest. Once our line counter is set up, we measure the time until a black line is observed by the camera. As a result, there is only one captured value per frame, where the yellow signal is the time of the interrupt and the blue is our v-sync pulse. This shows how only one line triggers the interrupt per frame.


We implement a **timer** that resets on every line and takes the output of the comparator as input. The comparator interrupts and halts on the first falling edge of the comparator output, the exact moment a black pixel is observed in the line. As a result, we are able to determine the position of the car relative to the black line by measuring the time taken to observe a black pixel. In order, the figures illustrate the pulse width when the black line is to the left of the car, when the line is directly in front of the car, and when the line is to the right of the car. The output of the timer is sent to an AND gate with the output of the line counter so the interrupt is triggered only once per frame. Otherwise, we would capture every line on every frame.
![alt text](https://github.com/kylmac/ELE302_Navigation_Control/blob/master/images/LeftDual.png)
![alt text](https://github.com/kylmac/ELE302_Navigation_Control/blob/master/images/CenterDual.png)
![alt text](https://github.com/kylmac/ELE302_Navigation_Control/blob/master/images/RightDual.png)

The blue signal is our non-inverted h-sync pulse and the yellow signal is the pulse at the interrupt. We time from the rising edge of the first blue pulse until the rising edge of the yellow signal. This is an effective method for determining the car’s position relative to the black line. The clock for both the timer and counter is set at 1MHz. The clock speed determines the resolution of our measurement. If the clock cycle is longer than the time taken to observe the black line, we can not calculate the car’s position. The period of one line is about 64μs and there are 320 pixels per line. This implies a clock frequency greater than 5MHz would be redundant, as a clock cycle period that is less than the time to read a pixel does not provide any more information than one that is the same as one pixel. Using this knowledge, we found that a 1MHz clock frequency was sufficient to time the signal.



#### Control Code
Our software reads the time taken to observe a black line and writes a PWM signal to the servo motor based on the the measurement. Once the interrupt is triggered by the black line, the time taken to observe the line is read in and compared with a target value of 36. The target value is acquired by lining up the black line with the center of the car and is measured in units of clock cycles. With a clock frequency of 1MHz, 36 clock cycles implies 36μs until the black lines is observed. 

The measurement is then input into a PD loop to correct any error between the target and measured value. We found that a K<sub>p</sub> value of 1.35 and K<sub>d</sub> of 0.01 was good enough to track the line within the time constraints. It is likely possible to complete the course using only proportional control. Once we calculated our error weighted by our tuning parameters, we shifted the error by 155 before writing to the PWM unit.




#### Top Design: Sending the PWM Signal
We send the appropriate **PWM** signal to the servo motor by writing a duty cycle to the input of the PWM unit on the top design, as shown in Figure. 9. The primary design decision made is determining the clock speed. We chose a clock speed of 100MHz to have greater precision when writing a duty cycle to the PWM unit. We first set the period of the PWM unit to 20ms, then found the required PWM input to transmit a signal with pulse width 1ms and 2ms. A pulse width of 1ms turns the wheels to the left while a pulse width of 2ms turns the wheels to the right. Through calibration, we found that writing a value of 117 to the PWM turned the wheels as far as possible to the left and 187 all the way to the right. We also found that writing 155 to the PWM keeps the wheels neutral, explaining our 155 offset in the PD control loop. The value we write to the PWM unit is an integer in the range of 0 and 1999 that sets the duty cycle of the signal to the fraction of the input integer over 1999, the period of the signal. With this, we were successfully able to send a PWM signal based on the error calculated through control.

![alt text](https://github.com/kylmac/ELE302_Navigation_Control/blob/master/images/PWM.png)
