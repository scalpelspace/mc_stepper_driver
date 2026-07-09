# mc_stepper_driver

Low level communication drivers for the stepper motor controller board running
the [mc_stepper](https://github.com/scalpelspace/mc_stepper) firmware.

---

<details markdown="1">
  <summary>Table of Contents</summary>

<!-- TOC -->
* [mc_stepper_driver](#mc_stepper_driver)
  * [1 Overview](#1-overview)
  * [2 CAN Bus Drivers](#2-can-bus-drivers)
  * [3 CAN Bus Messages](#3-can-bus-messages)
    * [3.1 Message Table](#31-message-table)
    * [3.2 Muxed Setpoint (`command_stepper`)](#32-muxed-setpoint-command_stepper)
<!-- TOC -->

</details>

---

## 1 Overview

CAN drivers are included in the [mc_stepper_driver.h](mc_stepper_driver.h) file
for simple implementation.

This repository defines the CAN **protocol contract** for MC Stepper: message
IDs, DLCs, signal layouts and scaling. The source of truth is the DBC file:
[can_mc_stepper.dbc](can_mc_stepper.dbc).

> **Note:** Behavioural semantics (controls state machine, safety gating of
> config commands, telemetry rates, fault handling) are owned by the
> [mc_stepper](https://github.com/scalpelspace/mc_stepper) firmware and are
> documented there.

---

## 2 CAN Bus Drivers

CAN drivers are implemented via the [
`can_driver`](https://github.com/scalpelspace/can_driver) submodule.

All CAN IDs follow the ScalpelSpace node CAN ID standard defined by
`can_driver`: the 11-bit classic CAN ID is split into a 6-bit `message_id` and
a 5-bit `node_id`, so a node's on-bus ID is its base message ID plus its
assigned node ID. Node IDs `0` (unassigned) and `31` (broadcast) are reserved,
and node IDs are assigned at runtime via the dynamic allocation protocol.

> See the [`can_driver` README](can_driver/README.md) for the full ID scheme,
> reserved ID ranges and the node ID allocation protocol.

---

## 3 CAN Bus Messages

### 3.1 Message Table

Direction is from the MC Stepper node's perspective: **TX** = transmitted by
MC Stepper, **RX** = received by MC Stepper.

| Base ID (hex) | Message                    | DLC | Direction | Description                                                                                                                               |
|--------------:|----------------------------|----:|-----------|-------------------------------------------------------------------------------------------------------------------------------------------|
|          0x20 | `state`                    |   5 | TX        | Periodic system state, MCU temperature, fault byte and TMC2209 status.                                                                    |
|          0x40 | `stallguard_event`         |   5 | TX        | Event frame on StallGuard stall detection edge.                                                                                           |
|          0x60 | `command_stepper`          |   6 | RX        | Control mode, enable, clear faults and muxed setpoint (see [3.2 Muxed Setpoint (`command_stepper`)](#32-muxed-setpoint-command_stepper)). |
|          0x80 | `command_stepper_zero`     |   1 | RX        | Software zero of the position reference.                                                                                                  |
|          0xA0 | `sensor`                   |   6 | TX        | Encoder raw count and unwrapped position (rad).                                                                                           |
|          0xC0 | `controls_diagnostic`      |   8 | TX        | Control loop internals: setpoints, position error and step rate command.                                                                  |
|          0xE0 | `motion_config_set`        |   7 | RX        | Set microstep size, max acceleration and max velocity.                                                                                    |
|         0x100 | `motion_config_get`        |   1 | RX        | Request motion configuration.                                                                                                             |
|         0x120 | `motion_config_response`   |   6 | TX        | Response to `motion_config_get`.                                                                                                          |
|         0x140 | `current_config_set`       |   6 | RX        | Set run current (mA) and IRUN/IHOLD/IHOLDDELAY scalers.                                                                                   |
|         0x160 | `current_config_get`       |   1 | RX        | Request current configuration.                                                                                                            |
|         0x180 | `current_config_response`  |   5 | TX        | Response to `current_config_get`.                                                                                                         |
|         0x1A0 | `controls_config_set`      |   8 | RX        | Set closed loop PID gains.                                                                                                                |
|         0x1C0 | `controls_config_get`      |   1 | RX        | Request PID gains.                                                                                                                        |
|         0x1E0 | `controls_config_response` |   6 | TX        | Response to `controls_config_get`.                                                                                                        |
|         0x200 | `stallguard_set`           |   3 | RX        | Enable StallGuard detection and set SGTHRS threshold.                                                                                     |
|         0x220 | `stallguard_get`           |   1 | RX        | Request StallGuard status.                                                                                                                |
|         0x240 | `stallguard_response`      |   5 | TX        | Response to `stallguard_get`: config, SG_RESULT and stalled flag.                                                                         |
|         0x260 | `datetime_set`             |   7 | RX        | Set the RTC date and time.                                                                                                                |
|         0x280 | `datetime_get`             |   0 | RX        | Request the RTC date and time.                                                                                                            |
|         0x2A0 | `datetime_response`        |   7 | TX        | Response to `datetime_get`.                                                                                                               |
|         0x2C0 | `rgb_led_set`              |   3 | RX        | Set the on-board RGB LED colour.                                                                                                          |
|         0x3C0 | `version_get`              |   0 | RX        | Request the firmware version.                                                                                                             |
|         0x3E0 | `version_response`         |   4 | TX        | Response to `version_get`: major, minor, patch and identifier.                                                                            |

### 3.2 Muxed Setpoint (`command_stepper`)

`command_stepper` carries a single 32-bit setpoint field (bytes 2..5)
multiplexed by the `control_mode` signal:

| `control_mode` | Active setpoint signal | Unit  |
|---------------:|------------------------|-------|
|              1 | `target_position_abs`  | rad   |
|              2 | `target_velocity`      | rad/s |
|              3 | `target_position_rel`  | rad   |
|              4 | `target_position_ol`   | rad   |

All setpoint signals are signed 32-bit with 0.001 scaling. A `control_mode` of
0 (or any unlisted value) carries no setpoint, allowing frames that only toggle
`enable` or `clear_faults`.
