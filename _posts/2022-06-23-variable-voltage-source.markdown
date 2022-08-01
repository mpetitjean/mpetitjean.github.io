---
layout: post
title:  "PWM-controlled variable voltage source using a LM317"
date:   2022-08-01 10:17:02 +0200
categories: hardware
---

# TL;DR

This post describes how to use a LM317 module to generate a voltage source that can be controlled by a microcontroller using PWM signals and a generic op-amp.

# The problem

One of my projects required me to generate a 1A power supply that could programmatically be adjusted between 2V and 9V by a microcontroller operating with 3.3V logic. 

The board embeds both 12V and 3.3V rails.

# The LM317 voltage regulator

The LM317 is an extermely widespread adjustable voltage regulator capable of supplying 1.5A over a voltage range of 1.25 to 37V (see [full datasheet][datasheet]).

The typical application circuit, as suggested in the datasheet, is shown below.

![Typical application circuit of the LM317](/img/lm317-base.png)

The output voltage is set by the value of the resistors R1 and R2, such that Vout = 1.25V x (1 + R2/R1). In an application where a constant voltage is required, the resistors can be set permanantly. The resistor R2 can also be replaced by a potentiometer to allow for manual adjustment.

The voltage generation can be better understood when looking at the functional block diagram of the LM317.

![LM317 functionnal block diagram](/img/lm317-block.png)

It can be seen that an internal op-amp with a constant input offset voltage of 1.25V is maintaining a constant 1.25V voltage difference between the OUTPUT and ADJUST pins. A higher R2 value will lead to a higher ADJUST voltage, and the OUTPUT voltage will be 1.25V more.

This constant 1.25V between the two pins can be exploited to programmatically change the output voltage.

[datasheet]: https://www.ti.com/lit/ds/symlink/lm317.pdf

# PWM control

Since my particular application requires an output voltage between 2V and 9V, the ADJUST pin needs to be freely set between 0.75V and 7.75V.

With a microcontroller that does not embed any DAC, this can be achieved by generating a 3.3V PWM signal which is then lowpass-filtered (*i.e.* averaged out), than amplified.

The resulting complete schematic is the following:

![Complete schematic with PWM control](/img/lm317-pwm.png)

The PWM signal is filtered to (roughly) DC by a low-pass filter with a 1Hz cut-off frequency. Depending on the ducty cycle of the PWM, the voltage at the input of the op-amp will range from 0 to 3.3V. It is then amplified a by the non-inverting op-amp with a gain of 2.3 to a range that roughly scales from 0 to 7.75V. Finally, the output voltage ranges from 1.25 to 9V.

Note that the resistor R4 is crucial for proper operation of the LM317 at no load. Indeed, as specified in the datasheet, a minimum load current of 3.5 to 10 mA is required to maintain the required output voltage. This current is flowing through resistor R4 when no other load is connected.
