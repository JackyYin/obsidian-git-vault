---
tags:
  - embedded
  - interrupt
---
> If you wanted to ensure that a program always caught the pulses from a rotary encoder, so that it never misses a pulse, it would make it very tricky to write a program to do anything else, because the program would need to constantly poll the sensor lines for the encoder, to catch pulses when they occurred. Other sensors have a similar interface dynamic, such as trying to read a sound sensor that detects a click, or an infrared slot sensor (photo-interrupter) that detects a coin drop. In all these situations, using an interrupt can free the microcontroller to do other work while still capturing the input.


## ISR (Interrupt Service Routine)

> ISRs are special kinds of functions that have unique limitations not shared by most other functions. An ISR cannot have any parameters and it should not return anything.


Refs:
- [https://docs.arduino.cc/learn/communication/uart/](https://docs.arduino.cc/language-reference/en/functions/external-interrupts/attachInterrupt/)