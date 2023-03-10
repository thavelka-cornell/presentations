Nexus Firmware Template - A Walk Through

Topics covered
**************

* Document Scope
* Overview of Dual-Core Applications - app coordination
* Environment and Tools - toolchain produces single binary
* Features Tested And Features Active
* Physical Set-Up
* Shared Memory Considerations
* Key Next Development Steps
* To Extend This Template


Scope of Nexus Firmware Walk-Through
************************************

This document aims to describe Nexus Template Firmware, a commit of Nexus gateway firmware which represents the integration of most key programmatic device features which "live" in the Nexus Gateway's primary micro-controller unit.  Firmware toolchain and development enrinment are mentioned by name.  Prototype board set-up is described to support physical running and observation of firmware on target hardware.  Dual-Core, dual application design choices and constraints are desribed.  Next development steps, and "common path" or likely ways to extend firmware functionality are briefly described.


Dual-Core Application Overview
******************************

Nexus Gateway incorporates the NXP LPC55S69 dual-core micro-controller (MCU).  This MCU provides two 32-bit ARM based cores which share memory, on-chip peripherals analog and digital, and which can communicate with each other via a simple inter-processor mailbox (IPM) facility.  Firmware team has taken earlier Pulse and Pulse Pro firmware work, and added new work in firmware, to a dual-application project which runs distinct, coordinated code on both these cores.


Firmware Dev Environment And Tools
**********************************

Nexus firmware development from middle 2022 to present 2023 Q1 takes place on Linux work stations.  A recent Zephyr RTOS toolchain forms basis of our build and debugging facilities.  Cmake configures our project during each build, calls compilers, debuggers and supporting tools.  Our C library implementation choice is 'newlibc'.  A Python based facility named `west` helps manage our Zephyr application's third party repositories, which are mostly defined by and effectively a part of the Zephyr RTOS Project.


Nexus Gateway Features
**********************

Gateway Features which we have implemented, tested, and some of which are active in the firmware template are in tabular format:

+---------+-----------+------------+---------------------------------------------+
| Feature |  Tested   |   Active   |  Notes                                      |
+=========+===========+============+=============================================+
|   UART  |    yes    |     yes    |  demo and diagnostics from each core        |
+---------+-----------+------------+---------------------------------------------+
|   I2C   |    yes    |     no     |                                             |
+---------+-----------+------------+---------------------------------------------+
|   SPI   |    yes    |     yes    |  via sensor communications                  |
+---------+-----------+------------+---------------------------------------------+
|   GPIO  |    yes    |     yes    |  via LED output, and via pulse counting     |
+---------+-----------+------------+---------------------------------------------+
|   ADC   |    yes    |     no     |                                             |
+---------+-----------+------------+---------------------------------------------+
|   PWM   |    yes    |     no     |                                             |
+---------+-----------+------------+---------------------------------------------+
| mailbox |    yes    |     yes    |  provides application coordination          |
+---------+-----------+------------+---------------------------------------------+



