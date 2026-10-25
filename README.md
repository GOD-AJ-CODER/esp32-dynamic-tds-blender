# Smart Dual-Loop RO Water Optimizer (ESP32)
![Hardware](https://img.shields.io/badge/Hardware-ESP32_WROOM-blue)
![Logic](https://img.shields.io/badge/Logic-Dual_Proportional_Feedback-emerald)
![Dashboard](https://img.shields.io/badge/Dashboard-100%25_Offline_Edge-darkgray)

## The Problem (And Why Current RO Systems Fail)

Standard RO purifiers have two massive issues:

1. **Massive Water Wastage:** A normal domestic RO system only recovers about 25% to 30% of water. For every 1 liter of clean water you drink, 3 liters are flushed straight down the drain. If a family consumes 10 liters a day, that is roughly 30 liters wasted daily—around 900 liters every single month. Because of how severe this is, the **National Green Tribunal (NGT)** and the **Bureau of Indian Standards (BIS)** stepped in to mandate at least a 60% recovery rate on newer systems.

2. **The "Dead Water" Issue & Static Bypass Failure:** RO membranes are too aggressive. They strip 90% to 99% of all minerals out of the water, dropping the TDS from 300+ ppm down to 10–20 ppm. Water this pure tastes flat, turns slightly acidic, and lacks essential calcium and magnesium. According to **WHO guidelines** and **BIS (IS 10500:2012)**, healthy drinking water should sit between **50 ppm and 150 ppm** (with **75 ppm** being the ideal sweet spot).

### How companies fix this today (and why it breaks):
* **For the Waste (Problem 1):** Purifiers use a dumb, static plastic plug called a **Flow Restrictor (FR)**. It restricts flow by a fixed amount 100% of the time, dumping massive amounts of water regardless of whether your tap water is relatively clean (150 TDS) or extremely hard (1,000+ TDS).
* **For the Minerals (Problem 2):** Today, purifier brands put a mechanical "TDS Controller" (a bypass valve) inside the machine. It takes a small stream of raw, unfiltered water and mixes it directly into the 100% pure RO water to bring the TDS back up to a drinkable ~75 ppm.

**The catch?** That valve is just a dumb manual screw. A technician comes to your house, sets the screw once with a screwdriver, and leaves. The moment your municipal supply or groundwater hardness changes the next day, the mixing ratio goes completely wrong—giving you either mineral-dead water or overly hard water.

---

## What We Built (The Dual-Loop Digital Logic Solution)

We replaced both the fixed flow restrictor and the manual TDS screw with an **Automated Dual-Loop Proportional Feedback Controller** running simultaneously on a single ESP32.

### Loop A: Automated Mineral Blending (The Drinking Side)
- The system continuously reads the TDS of the final blended drinking water.
- It compares the live reading against our **75 ppm WHO target** (with an allowable deadband of **65 to 85 ppm**).
- **If TDS drops below 65 ppm:** The water is over-filtered. The ESP32 calculates the error and opens the bypass valve proportionally to mix in more mineral-rich water.
- **If TDS goes above 85 ppm:** The water is too hard. The ESP32 closes the bypass valve proportionally, forcing more water through the RO membrane.
- **If TDS is between 65 and 85 ppm:** The system holds the valve steady.

### Loop B: Adaptive Reject Throttling (The Waste Side)
- The system simultaneously monitors the salt concentration (TDS) of the waste water heading to the drain.
- **When Reject TDS is LOW (< 500 ppm):** The ESP32 pinches the reject line tighter. Less waste escapes, building backpressure and forcing more clean water through the membrane (Saving Water).
- **When Reject TDS is HIGH (> 900 ppm):** The brine is dangerously concentrated. The ESP32 fully opens the waste line to flush the system and carry away the salt buildup before it crystallizes and ruins the RO membrane (Protecting the Filter).

---

## Live Telemetry Dashboard

Instead of hosting a separate website, the entire frontend dashboard is packed into a single `index.html` file and stored directly inside the **ESP32's PROGMEM flash memory**. When the ESP32 connects to Wi-Fi, you type its IP address into any browser and it serves the live UI directly.

<img width="1856" height="980" alt="Screenshot From 2026-10-21 20-49-43" src="https://github.com/user-attachments/assets/10a412d7-9a7d-4445-bedd-f1110bc05d0f" />

<img width="1856" height="980" alt="Screenshot From 2026-10-21 20-49-26" src="https://github.com/user-attachments/assets/430182c9-d196-4e12-ba3b-f2e0ea9d222a" />

### Dashboard Features:
- Live Chart.js telemetry tracking both **Output TDS (ppm)** against the 75 ppm target line, and **Reject Brine TDS (ppm)**.
- Dual visual indicators for the **Bypass Valve (%)** and the **Reject Throttle Valve (%)**.
- Real-time State Feedback Terminal showing the exact mathematical rules the ESP32 is executing across both loops at that moment.
- Instant toggle between **Autonomous Closed-Loop Mode** and **Manual Override** (allowing you to lock the valve percentages using sliders).

---

## Hardware Architecture & Pin Breakdown

Because Tinkercad doesn't support 32-bit ESP32 boards, we built and verified the circuit topology on **Wokwi**.

<img width="908" height="472" alt="Screenshot From 2026-10-25 15-28-03" src="https://github.com/user-attachments/assets/6068f168-a463-42d4-8a80-48ca91816102" />


## Why We Chose This Specific Hardware

- **ESP32 vs. Standard Arduino Uno:** We deliberately chose the ESP32 over a classic Arduino Uno. An Uno is an 8-bit board with no built-in Wi-Fi and only 2KB of RAM. The ESP32 is a 32-bit powerhouse with native Wi-Fi, meaning it can host our entire HTML/JS web dashboard directly from its internal flash memory without needing an external cloud server, Vercel, or an SD card.

- **Optocoupled Relay / MOSFET Drivers:** Microcontrollers run on a highly sensitive 3.3V logic rail. Our physical water valves require 12V and high current to move the internal mechanics. If we wired them together directly, the inductive back-voltage would instantly fry the ESP32. The relays act as electrical isolation barriers between the low-voltage brain and the high-voltage plumbing. 

- **Analog TDS Probes:** We needed sensors that give a direct, instantaneous analog voltage correlated to the water's electrical conductivity. This allows us to use the ESP32's internal ADCs and apply a 10x oversampling algorithm in C++ to smooth out any electrical pump noise before calculating the exact ppm for both lines.

### Deep Dive: How the Pins & Relays Actually Work

- **GPIO 34 & GPIO 35 (Inputs — Analog TDS Sensors):** 
  These are wired to the signal pins of our two Analog TDS sensors. On the ESP32, these pins belong to **ADC1 (Analog-to-Digital Converter 1)**. They read the raw 0V to 3.3V analog signals coming from the drinking water and waste water probes and convert them into 12-bit digital numbers (`0` to `4095`). To stop electrical pump noise from messing up the readings, our code takes 10 rapid samples in a row and averages them before converting the voltage into PPM.

- **GPIO 25 & GPIO 26 (Outputs — Dual Valve Control via PWM):** 
  These are our actuator output pins. Once the ESP32 calculates the exact percentage the bypass and reject valves need to be open, these pins output 5 kHz **PWM (Pulse Width Modulation)** signals to drive the actuators.

- **The Relay / Driver Modules (Why they are there & how they work):** 
  The ESP32 runs on tiny **3.3V logic**, while a physical motorized valve needs **12V** and high current to move. 
  * **Internal Mechanism:** The relays act as an electrically isolated bridge. Their input side listens to the 3.3V signals from GPIO 25 and 26. When a pin goes HIGH, a small electromagnetic coil inside the relay magnetizes and physically pulls an internal mechanical contact shut (making an audible *"click"* sound). That switch closes the separate 12V circuit to open/modulate the water valves, keeping the high-voltage plumbing completely isolated from our 3.3V microcontroller.
