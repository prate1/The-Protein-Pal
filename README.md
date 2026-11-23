# The-Protein-Pal

ECE and Hardware Electronics track repository for **The Protein Pal** — an automated, contactless protein shake vending and compounding kiosk (**TPP India / Hexcube Pvt. Ltd.**).

This repository documents the system-level hardware architecture, multi-rail power distribution, I2C peripheral bus topology, sensor telemetry integration, firmware orchestration, and root-cause hardware debugging validated across **140+ end-to-end automated dispensing cycles**.

> **Confidentiality Notice:** Proprietary mechanical CAD, commercial recipes, and exact production pin mappings are omitted to respect company intellectual property guidelines. Code modules represent generalized architectural patterns.

---

## System Block Diagram


+-------------------------------------------------------------------------+

|                  MULTI-RAIL POWER ARCHITECTURE (230V AC NC E-Stop)      |
|  [24V PSU (350W)] ---> Stepper Drivers (TB6600 / TMC2209) & DC Bus      |
|  [12V PSU (75W)]  ---> Peristaltic/Water Pumps, Solenoids, Actuators    |
|  [6V Buck Conv.]  ---> Isolated Clamping Servos (1000uF + 0.1uF Decoup.) |
|  [5V / 3.3V Buck] ---> Raspberry Pi 4B, ESP32, I2C Logic & Pull-ups     |
+------------------------------------+------------------------------------+
                                     |
                                     v
+-------------------------------------------------------------------------+

|               HOST CONTROLLER: Raspberry Pi 4B (Linux OS)               |
|           Klipper Firmware + Moonraker REST API (localhost:7125)        |
+-------+----------------------------+----------------------------+-------+

        | USB / UART                 | I2C Bus                    | GPIO
        v                            v                            v
+-------------------+   +-------------------------+   +-------------------+

| MOTION BOARDS     |   | I2C PERIPHERAL EXPANDERS|   | SENSORS & TELEM.  |
| Dual BTT Octopus  |   | 0x20: XL9535 (16-Ch Rly)|   | Optical / IR /    |
| Pro               |   | 0x21: 2-Ch Water Relay  |   | Proximity Limits  |
| - 11x NEMA17      |   | 0x41: PCA9685 (12-bit   |   | ESP32 + HX711     |
|   (TMC2209 UART)  |   |       PWM Servo Driver) |   | Load Cell Module  |
| - 1x NEMA23 &     |   +-------------------------+   +-------------------+
|   1x NEMA34       |
|   (TB6600)        |
+-------------------+




## Hardware Architecture & Subsystems

* Host Controller & Orchestration:** **Raspberry Pi 4B** running Linux, **Klipper** firmware, and the **Moonraker REST API** (`localhost:7125`) to coordinate multi-stage dispensing, mixing, and fault recovery.
* Multi-Axis Motion Control:** Dual **BTT Octopus Pro** boards managing **13 stepper motors**—11× NEMA 17 motors driven via **TMC2209** UART drivers and high-torque NEMA 23 / NEMA 34 axes driven via external **TB6600** drivers.
* I2C Peripheral Bus (`0x20`, `0x21`, `0x41`):**
  * `0x20` (XL9535 16-Channel I2C Relay):** Switches 10 peristaltic syrup pumps and dual lid pick-and-press linear actuators.
  * `0x21` (2-Channel I2C Relay):** Controls the high-flow main water dispensing pump.
  * `0x41` (PCA9685 16-Channel PWM Driver):** Controls opposing high-torque clamping servos (MG996R / MG985) mounted on the rotational mixing assembly via a multi-channel **slip ring**.
* **Gravimetric & Optical Sensing:** **ESP32** interfaced with **HX711** load-cell amplifiers for weight verification, paired with ~10 industrial optical, IR, and proximity sensors wired to Raspberry Pi GPIOs for stage verification.
* Safety & Power Integrity:** Star-grounding topology, flyback suppression across inductive pump/solenoid loads, bulk decoupling (`1000 µF` electrolytic + `0.1 µF` ceramic) on the 6V servo rail, and a **normally-closed (NC) 230V AC E-Stop relay** cutting primary mains power on emergency trigger.


## Hardware Failure Analysis & Root-Cause Debugging

Real-world hardware bring-up surfaced several electromechanical and electrical faults across 140+ test cycles. Below is the root-cause analysis and mitigation implemented for each:

1. *Controller Board Failure & Stall Overcurrent Protection (BTT Octopus Pro):**
   * *Symptom:* One BTT Octopus Pro controller board suffered catastrophic hardware failure during testing following a mechanical hard-stop event (the cup dispenser carriage stalled against the end of the guide rods for >15 seconds, producing an audible electrical/mechanical fault), alongside potential **TMC2209** driver pin shorting.
   * *Root-Cause Analysis:* Sustained mechanical stall forced continuous peak current through the driver stage without endstop cutoff, combined with potential back-EMF/short-circuit stress across the driver socket.
   * *Mitigation:* Enforced strict hardware endstop limits (`STOP_ON_ENDSTOP`), added software travel bounds (`min`/`max` soft limits) in Klipper config, and instituted pre-power continuity/pinout verification protocols before seating TMC2209 modules.

2. **Intermittent Power & Signal Faults from Crimp & WAGO Terminals:**
   * *Symptom:* Random communication dropouts, erratic sensor triggers, and intermittent actuator power loss during multi-axis vibration cycles.
   * *Root-Cause Analysis:* High-frequency mechanical vibration from NEMA 34/23 steppers and the mixing station caused micro-looseness in improperly seated wire crimps and WAGO lever connectors, increasing contact resistance and causing voltage drops.
   * *Mitigation:* Re-terminated harnesses with calibrated ratcheting crimpers, performed pull-tests on every ferrule/crimp pin, added strain-relief zip-tie anchors near moving axes, and isolated high-vibration harness runs.

3. **Asymmetric Servo Travel & Stall Jitter Under 330 ml Payload (`PCA9685`):**
   * *Symptom:* Opposing clamp servos (MG996R & MG985) opened at unequal angles and exhibited severe mechanical vibration/jitter while clamping a loaded **330 ml cup**.
   * *Root-Cause Analysis:* Calling `pca.deinit()` previously left PWM control lines floating and susceptible to EMI, while commanding aggressive closing speeds and over-driven pulse widths (`µs`) caused the two opposing servos to fight each other mechanically (stall current draw + 6V rail droop).
   * *Mitigation:* Retained active PWM hold state (eliminating line float), tuned independent microsecond endpoints (`LEFT_OPEN`/`CLOSED` vs. `RIGHT_OPEN`/`CLOSED`) to equalize physical flap angles, reduced closing angular velocity via a **25-step synchronized trajectory ramp (`ramp_both`)**, and adjusted holding pulse width to balance grip torque against stall vibration.

4. **Parallel Non-Blocking Motion (`SYNC=0`) & Power-Cycle Recovery:**
   * *Mitigation:* Overlapped non-blocking secondary moves (`MANUAL_STEPPER ... SYNC=0`) with primary blocking trajectories to cut cycle time, added state-variable re-syncing (`SET_GCODE_VARIABLE` + `SET_POSITION`) to prevent leadscrew drift on reboot, and implemented a **1-second reverse-polarity pulse** on syrup pumps to stop nozzle dripping.



## Repository Structure
* `/examples/synchronized_servo_clamp.py` — Synchronized dual-servo trajectory ramping over I2C (`PCA9685`) with active PWM holding.
* `/examples/moonraker_cycle_orchestrator.py` — Moonraker REST API state-machine template demonstrating parallel `SYNC=0` stepper execution.
