+++
title = 'System Topology'
date = 2024-01-27
+++

<link rel="stylesheet" href="/css/style.css">

Every system has a **topology**. The high level view of what sub-systems exist and how they interact.

*The Monster* will be segmented into three sub-systems:
1. Power
2. Communication
3. Control

These are connected by a shared bus and can inform/control each other accordingly.

## Power

A self-starting digital power system with primary and auxiliary output.

Likely an STM32G071RB with two half-bridges.

### Responsibilities
- Regulating voltages for logic and gate driving
- Regulating voltages for external modules
- System power state (ex. power button)
- Battery voltage measurement

## Communication

A BLE module with OTA DFU capability running the Monster BLE stack to interact with the application layer.

### Responsibilities
- Monitoring
- Control
- Logging
- Error reporting
- DFU
  - Self
  - Other sub-systems

## Control

*The Monster's* fangs. Runs the FOC/balance loop.

### Responsibilities
- FOC
- Balance
- Battery current measurement

*Shortest list and yet doing the most work!*

---

We'll learn more about the details as they arise, but it's good to have this high level overview to refer to.

Things will get much more interesting once we enter the initial implementation stage (testing).