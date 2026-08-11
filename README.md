# The-Protein-Pal
ECE and Hardware Electronics track repository for The Protein Pal — automated screenless, kiosk for contactless protein shake vending machine.

#The Protein Pal — ECE & Hardware Electronics Team

Welcome to the hardware and electronics repository for **The Protein Pal**, an automated, screenless, contactless protein vending and compounding kiosk. This repository houses the hardware architecture, pin configurations, sensor telemetry logic, and firmware documentation for our V1 functional prototype.



# Hardware description 

Our ECE team owns the complete physical execution, power distribution, and telemetry backbone of the machine. The core subsystems include:

* Main Processor(The brain): Raspberry Pi 4 (2GB RAM) running Linux OS, handling Python-based sensor polling, GPIO motor control, local logging, and MQTT cloud communication.
* Gravimetric Powder Dosing: NEMA Stepper Motor with auger screw, driven via A4988/TMC2209 drivers, coupled with "HX711 load cell amplifiers " for real-time gram-level precision.
* Fluid Control & Dispensing: Food-grade solenoid valves, 12V diaphragm pumps, and "YF-S201 Hall-effect flow meters" for exact 300 ml water volume measurement.


# Block Diagram of system 
[ Power Management ] (12V / 5V DC Rails)
│
▼
[ Sensors Subsystem ] ──► [ Raspberry Pi 4 (2GB) ] ──► [ Actuator Subsystem ]

Load Cells (HX711)      (Linux / Python / MQTT)     - Stepper Drivers & Augers

Flow Meters (Water)                                 - Solenoids & Water Pumps

Cup IR & Door Interlocks                            - LED Ring & Buzzer

[ Power Management ] (12V / 5V DC Rails)
       │
       ▼
[ Sensors Subsystem ] ──► [ Raspberry Pi 4 (2GB) ] ──► [ Actuator Subsystem ]
- Load Cells (HX711)      (Linux / Python / MQTT)     - Stepper Drivers & Augers
- Flow Meters (Water)                                 - Solenoids & Water Pumps
- Cup IR & Door Interlocks                            - LED Ring & Buzzer
