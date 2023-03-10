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



Physical Set-Up
***************

::
  1   + ----------------------------------+
  1   |                                   |
  1   |   --------          --------      |
  1   |                                   |
  1   |   Click 2           Click 4       |
  1   |                                   |
  1   |   --------          --------      |
  1   |                                   |
  1   |                                   |
  1   |       RT G                        |
  1   |       XX N                        |
  1   |       11 D                        |
  1   |   --------          --------      |
  1   |                                   |
  1   |   Click Cell        Click 3       |
  1   |                                   |
  1   |   --------          --------      |
  1   |                     G3||||        |      +---------+
  1   |                     NV++++================  KX132  |
  1   |                     D3            |      +---------+
  1   |                                   |
  1   |        RX_2 TX_2                  |
  1   + ----------------------------------+



Shared Memory Considerations
****************************

Device Tree Source and Zephyr allow us to partition memory resources and share them in mutually exclusive and inclusive ways.  By design we share the LPC55S69's static random access memory in both ways.

Static RAM that's used for application stack and heap space, and for Zephyr's static RAM needs must be given to each processor core in a mutually exclusive way.  In device tree source code and for Zephyr RTOS, there is a special node named `chosen`.  The node named 'chosen' does not represent hardware (1).  Other nodes whose names appear as properties in this chosen node can be thought of as being default or first choice hardware resources for Zephyr.

In the case of SRAM the chosen SRAM node represents the memory which our C programs will utilize in traditional, high level C language ways.  For practical purposes in a Zephyr app, statically and locally declared variables live in this memory range of the memory.

Memory which lies outside each Nexus application's chosen nodes can still be accessed.  This "not chosen" memory will be visible to each core on which device tree enables that memory just as a typical enabled device tree node.  When multiple cores have app's whose device tree enables like memory ranges, those cores will share that memory.

The compiler effectively does not see memory outside of Zephyr's chosen-qualifed memory ranges, so we must access this memory through pointers only.  We don't get to declare variables, and we must keep track of the addresses of interest in this shared memory.  In a dual-core project where coordination of data is needed between applications, we must device ways to inform both applications of the meanings -- the addresses -- of data which lives in shared memory.  We must also account for and design against race conditions in all shared memory accesses.

The LPC55S69 Inter-Processor Mailbox (IPM) provides us a way to effectively share memory between applications running in parallel on two cores.


Footnote:
(1)  https://elinux.org/Device_Tree_Usage#Special_Nodes


Device Tree Source excerpt
**************************

From `zephyr/dts/arm/nxp/nxp_lpc55S6x_common.dtsi`:
::
 54         /* lpc55S6x Memory configurations:
 55          *
 56          * RAM blocks SRAM0 through SRAM4 are contiguous address ranges
 57          *
 58          * LPC55S66: 144KB RAM, RAMX: 32K, SRAM0: 32K
 59          * LPC55S69: 320KB RAM, RAMX: 32K, SRAM0: 64K, SRAM1: 64K,
 60          *                      SRAM2: 64K, SRAM3: 64K, SRAM4: 16K
 61          */
 62         sram0: memory@20000000 {
 63                 compatible = "mmio-sram";
 64                 reg = <0x20000000 DT_SIZE_K(64)>;
 65         };
 66 
 67         sram1: memory@20010000 {
 68                 compatible = "mmio-sram";
 69                 reg = <0x20010000 DT_SIZE_K(64)>;
 70         };
 71 
 72         sram2: memory@20020000 {
 73                 compatible = "mmio-sram";
 74                 reg = <0x20020000 DT_SIZE_K(64)>;
 75         };
 76 
 77         sram3: memory@20030000 {
 78                 compatible = "mmio-sram";
 79                 reg = <0x20030000 DT_SIZE_K(64)>;
 80         };
 81 
 82         sram4: memory@20040000 {
 83                 compatible = "mmio-sram";
 84                 reg = <0x20040000  DT_SIZE_K(16)>;
 85         };

We resize SRAM1 partition in our project's `nexus.dtsi` source file:
::
 &sram1 {
     compatible = "mmio-sram";
     reg = <0x20000000 DT_SIZE_K(128)>;
 };

Our overlay changes for core number 1:
::
 chosen {
     zephyr,sram = &sram0;
     .
     .
     .
 }

 &sram2 {
     status = "disabled";  // disabled because it's combined with SRAM1 paritition
 };

 &sram3 {
     status = "disabled";  // disabled to respect other core's application memory
 };

Our overlay changes for core number 2:
::
 chosen {
     zephyr,sram = &sram3;
     .
     .
     .
 }

 &sram2 {
     status = "disabled";
 };

 &sram0 {
     status = "disabled";
 };

Take-away points from our DTS overlays:

  *  core 0 choses SRAM0 partition for its application and Zephyr dedicated RAM
  *  core 0 disables SRAM2 and SRAM3 partitions

  *  core 1 choses SRAM3 partition for its application and Zephyr dedicated RAM
  *  core 1 disables SRAM2 and SRAM0 partitions

  *  dual-core project wide dtsi file re-sizes shared SRAM1 partition from 64kB to 128kB


Key Next Development Steps
**************************

*  Test and validate inter-processor mailbox mutex
*  Create first draft design description of Nexus app-to-app messaging protocol
*  Implement external flash module
*  Enable ADC thread or k_work instance
*  Implement and enable temperature sensing


To Extend This Template
***********************

In Nexus Zephyr based applications two of the more likely places to extend functionality lie in the creation of a new thread, or a new k_work structure.  Threads will typically run for the full duration of a app's execution time, and threads often implement stateful amd more complex activities.  Threads also tend to incur more and longer term memory use compared with instances of k_work structures, which are passed to the Zephyr kernel for processing.

A k_work structure in Zephyr gets passed to a Zephyr thread which executes application tasks on request, with or without delays, and as soon as scheduling and resources permit.  A k-work structure use is typically good for single-shot type tasks, tasks which can run in a shorter period of time and do not themselves require storage or statefulness outside of their run times.

In summary, Zephyr threads and "kernel work" structure instances are the more likely ways to extend Nexus firmware template functionality.




