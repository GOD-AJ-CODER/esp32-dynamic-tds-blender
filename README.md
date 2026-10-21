# Smart Closed-Loop TDS Water Optimizer (ESP32)

## The Problem (And Why Current RO Systems Fail)

Standard RO purifiers have two massive issues:

1. **Massive Water Wastage:** A normal domestic RO system only recovers about 25% to 30% of water. For every 1 liter of clean water you drink, 3 liters are flushed straight down the drain. If a family consumes 10 liters a day, that is roughly 30 liters wasted daily—around 900 liters every single month. Because of how severe this is, the **National Green Tribunal (NGT)** and the **Bureau of Indian Standards (BIS)** stepped in to mandate at least a 60% recovery rate on newer systems.
2. **The "Dead Water" Issue & Static Bypass Failure:** RO membranes are too aggressive. They strip 90% to 99% of all minerals out of the water, dropping the TDS from 300+ ppm down to 10–20 ppm. Water this pure tastes flat, turns slightly acidic, and lacks essential calcium and magnesium. According to **WHO guidelines** and **BIS (IS 10500:2012)**, healthy drinking water should sit between **50 ppm and 150 ppm** (with **75 ppm** being the ideal sweet spot).

### How companies fix this today (and why it breaks):
Today, purifier brands put a mechanical "TDS Controller" (a bypass valve) inside the machine. It takes a small stream of raw, unfiltered water and mixes it directly into the 100% pure RO water to bring the TDS back up to a drinkable ~75 ppm. 

**The catch?** That valve is just a dumb manual screw. A technician comes to your house, sets the screw once with a screwdriver, and leaves. The moment your municipal supply or groundwater hardness changes the next day, the mixing ratio goes completely wrong—giving you either mineral-dead water or overly hard water.

---

## What We Built (The Pure Digital Logic Solution)

We replaced that manual screw with an **Automated Closed-Loop Proportional Feedback Controller** running on an ESP32. 

* The system continuously reads the TDS of the final blended drinking water.
* It compares the live reading against our **75 ppm WHO target** (with an allowable deadband of **65 to 85 ppm**).
* **If TDS drops below 65 ppm:** The water is over-filtered. The ESP32 calculates the error and opens the bypass valve proportionally to mix in more mineral-rich water.
* **If TDS goes above 85 ppm:** The water is too hard. The ESP32 closes the bypass valve proportionally, forcing more water through the RO membrane.
* **If TDS is between 65 and 85 ppm:** The system holds the valve steady.

---

## Live Telemetry Dashboard

Instead of hosting a separate website, the entire frontend dashboard is packed into a single `index.html` file and stored directly inside the **ESP32's PROGMEM flash memory**. When the ESP32 connects to Wi-Fi, you type its IP address into any browser and it serves the live UI directly.

<img width="1856" height="980" alt="Screenshot From 2026-10-21 20-49-43" src="https://github.com/user-attachments/assets/10a412d7-9a7d-4445-bedd-f1110bc05d0f" />

<img width="1856" height="980" alt="Screenshot From 2026-10-21 20-49-26" src="https://github.com/user-attachments/assets/430182c9-d196-4e12-ba3b-f2e0ea9d222a" />



### Dashboard Features:
* Live Chart.js telemetry tracking Output TDS (ppm) against the 75 ppm target line and Bypass Valve (%).
* Real-time State Feedback Terminal showing the exact mathematical rule the ESP32 is executing at that moment.
* Instant toggle between **Autonomous Closed-Loop Mode** and **Manual Override** (allowing you to lock the valve percentage using a slider).

---

## Hardware Architecture & Pin Breakdown

Because Tinkercad doesn't support 32-bit ESP32 boards, we built and verified the circuit topology on **Wokwi**.

<img width="811" height="740" alt="Screenshot From 2026-10-21 20-59-50" src="https://github.com/user-attachments/assets/da5088ae-2258-4e87-ac01-bfaafbae6265" />


### Deep Dive: How the Pins & Relay Actually Work

* **GPIO 34 (Input — Analog TDS Sensor):** 
  GPIO 34 is wired to the signal pin of our Analog TDS sensor. On the ESP32, GPIO 34 belongs to **ADC1 (Analog-to-Digital Converter 1)**. GPIO 34 reads the raw 0V to 3.3V analog signal coming from the water probe and converts it into a 12-bit digital number (`0` to `4095`). To stop electrical pump noise from messing up the readings, our code takes 10 rapid samples in a row and averages them before converting the voltage into PPM.

* **GPIO 25 (Output — Valve Control via PWM):** 
  GPIO 25 is our actuator output pin. Once the ESP32 calculates the exact percentage the bypass valve needs to be open, GPIO 25 outputs a 5 kHz **PWM (Pulse Width Modulation)** signal to drive the actuator.

* **The Relay / Driver Module (Why it's there & how it works):** 
  The ESP32 runs on tiny **3.3V logic**, while a physical water motorized valve needs **12V** and high current to move. If we wired a 12V valve directly to GPIO 25, the 12V back-feed would instantly fry the ESP32 chip. 
  * **Internal Mechanism:** The relay acts as an electrically isolated bridge. Its input side listens to the 3.3V signal from GPIO 25. When GPIO 25 goes HIGH, a small electromagnetic coil inside the relay magnetizes and physically pulls an internal mechanical contact shut (making an audible *"click"* sound). That switch closes the separate 12V circuit to open/modulate the water valve, keeping the high-voltage plumbing completely isolated from our 3.3V microcontroller.
