# mc_stepper_driver

Low level communication drivers for the Stepper Motor Controller dev board
running the [mc_stepper](https://github.com/scalpelspace/mc_stepper) firmware.

---

<details markdown="1">
  <summary>Table of Contents</summary>

<!-- TOC -->
* [mc_stepper_driver](#mc_stepper_driver)
  * [1 Overview](#1-overview)
  * [2 CAN Bus Drivers](#2-can-bus-drivers)
<!-- TOC -->

</details>

---

## 1 Overview

The Stepper Motor Controller dev board provides 2 direct communication
interfaces:

1. UART
2. CAN (classic)

**CAN** us supported natively through the software included in this driver
package.

CAN drivers are included in the [mc_stepper_driver.h](mc_stepper_driver.h) file
for simple implementation.

---

## 2 CAN Bus Drivers

CAN drivers are implemented via the [
`can_driver`](https://github.com/scalpelspace/can_driver) submodule.
