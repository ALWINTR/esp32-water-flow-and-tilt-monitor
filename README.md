# 💧 ESP32 Water Flow Rate & Structural Pipe Tilt Angle Monitor

[![GitHub Repository](https://img.shields.io/badge/GitHub-Repository-00f0ff?style=for-the-badge&logo=github&logoColor=white)](https://github.com/ALWINTR/esp32-water-flow-and-tilt-monitor)
[![Developer](https://img.shields.io/badge/Developer-Alwin_T_R-0284c7?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/alwintr)
[![Platform](https://img.shields.io/badge/Platform-ESP32_&_Flow_Sensors-38bdf8?style=for-the-badge&logo=espressif&logoColor=white)](https://github.com/ALWINTR)
[![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)](LICENSE)

Real-time liquid flow rate and pipe structural tilt angle monitoring station using ESP32, YF-S201, and SW-520D.

---

## 📌 System Architecture

A comprehensive dual-parameter telemetry station for pipeline infrastructure monitoring, combining high-speed Hall-effect pulse counting for real-time liquid flow measurement and tilt switch angle deviation detection for structural integrity alarms.

```
       ┌──────────────────────────────┐       ┌──────────────────────────────┐
       │   YF-S201 Water Flow Sensor  │       │     SW-520D Tilt Sensor      │
       │   (Hall Effect Interrupts)   │       │   (Angular Deviation Trip)   │
       └──────────────┬───────────────┘       └──────────────┬───────────────┘
                      │ Interrupt GPIO                       │ Digital GPIO
                      └───────────────► ┌─────────────┐ ◄────┘
                                        │    ESP32    │
                                        │  Controller │
                                        └──────┬──────┘
                                               ▼
                                        ┌─────────────┐
                                        │ OLED / IoT  │
                                        │  Telemetry  │
                                        └─────────────┘
```

---

## ⚙️ Mathematical Model & Flow Rate Calculation

The YF-S201 Hall-effect sensor outputs pulses proportional to water velocity:

$$	ext{Flow Rate } (L/	ext{min}) = rac{	ext{Pulses per Second}}{7.5}$$

$$	ext{Total Volume } (L) = \sum rac{	ext{Flow Rate} 	imes \Delta t}{60}$$

---

## 🔌 Circuit Pinout Table

| Sensor Pin | ESP32 GPIO Pin | Description |
| :--- | :--- | :--- |
| **YF-S201 Pulse Out** | GPIO 14 (Interrupt) | Hardware interrupt on rising pulse edge |
| **SW-520D Tilt Signal** | GPIO 27 (Pull-up) | Low when vertical, High when tilted > 15° |
| **OLED SDA / SCL** | GPIO 21 / GPIO 22 | I2C graphics datastream |
| **Warning Buzzer** | GPIO 4 | Acoustic alarm on pipe tilt or abnormal flow |

---

## 👨‍💻 Author

**Alwin T R** — Robotics & Automation Engineer  
- 💼 LinkedIn: [linkedin.com/in/alwintr](https://www.linkedin.com/in/alwintr)  
- 🌌 Portfolio: [alwintr.github.io](https://alwintr.github.io)  
- 💻 GitHub: [github.com/ALWINTR](https://github.com/ALWINTR)

---

## 📄 License

This project is licensed under the **MIT License** — see the [LICENSE](LICENSE) file for details.
