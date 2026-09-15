# 💬 Esp32 With Watter Flow Sensor And Tilt Sensor

- **Developer / User**: `Alwin T R`
- **Session ID**: `3f444d09-db7a-43de-bb9b-fb43c4de9777`
- **Started**: `2026-09-10T09:32:54Z`
- **Last Updated**: `2026-09-10T12:38:41Z`
- **Total Interaction Steps**: `164`

---

### 👤 Alwin T R (`2026-09-10T09:32:54Z`)

esp32 with watter flow sensor and tilt sensor

---

### 🤖 AI Engineering Assistant (`2026-09-10T09:32:54Z`)

Here is a complete guide and production-ready code to interface an **ESP32** with a **Water Flow Sensor (e.g., YF-S201)** and a **Tilt Sensor (e.g., SW-520D / Ball switch module)**.

---

### 1. Wiring & Circuit Connections

```
      +-----------------------------------------------------------+
      |                          ESP32                            |
      |                                                           |
      |   VIN (5V)       GND        GPIO 27            GPIO 26    |
      +------+------------+------------+------------------+-------+
             |            |            |                  |
      +------+------------+------------+---+       +------+-------+
      | VCC (Red)  GND (Blk)  Signal (Yel) |       | VCC  GND  DO |
      |                                    |       |              |
      |       Water Flow Sensor            |       |  Tilt Sensor |
      |           (YF-S201)                |       |   (SW-520D)  |
      +------------------------------------+       +--------------+
```

#### Pin Mapping Table

| Component | Component Pin | ESP32 Pin | Note |
| :--- | :--- | :--- | :--- |
| **Water Flow Sensor** | VCC (Red) | `VIN` (5V) or `3.3V` | YF-S201 runs accurately on 5V |
| | GND (Black) | `GND` | Common Ground |
| | Signal (Yellow) | `GPIO 27` | Pulse output (Hall effect) |
| **Tilt Sensor Module**| VCC | `3.3V` | |
| | GND | `GND` | Common Ground |
| | DO (Digital Out) | `GPIO 26` | Logic HIGH when tilted / upright |

> [!IMPORTANT]
> **5V Logic Safety**: If you power the YF-S201 flow sensor with **5V**, its pulse output signal is 5V. While many ESP32 pins are somewhat resilient, it is best practice to use a simple **voltage divider** (e.g., $1\,\text{k}\Omega$ and $2\,\text{k}\Omega$ resistors) to step down the signal to $\approx 3.3\text{V}$, or power the sensor directly from the `3.3V` rail if your flow rates are moderate.

---

### 2. Complete Arduino C++ Code

This code uses a **hardware interrupt** with `IRAM_ATTR` to count flow pulses accurately without missing counts, and a non-blocking `millis()` loop for periodic reporting.

```cpp
#include <Arduino.h>

// ---------------- Pin Definitions ----------------
const int FLOW_SENSOR_PIN = 27;  // Interrupt pin for Flow Sensor
const int TILT_SENSOR_PIN = 26;  // Digital pin for Tilt Sensor
const int LED_PIN         = 2;   // Onboard status LED

// ---------------- Flow Sensor Constants ----------------
// For YF-S201: Pulse frequency (Hz) = 7.5 * Q (L/min)
// Calibration factor = 7.5
const float CALIBRATION_FACTOR = 7.5;

// ---------------- Global Variables ----------------
volatile unsigned long pulseCount = 0;

float flowRate = 0.0;          // Liters per minute (L/min)
unsigned int flowMilliLitres = 0;
unsigned long totalMilliLitres = 0;

unsigned long previousMillis = 0;
const unsigned long INTERVAL = 1000; // Update interval in ms (1 second)

// ---------------- Interrupt Service Routine (ISR) ----------------
void IRAM_ATTR pulseCounterISR() {
  pulseCount++;
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  Serial.println("\n--- ESP32 Water Flow & Tilt Sensor Monitor ---");

  // Configure Pins
  pinMode(LED_PIN, OUTPUT);
  pinMode(TILT_SENSOR_PIN, INPUT_PULLUP);
  pinMode(FLOW_SENSOR_PIN, INPUT_PULLUP);

  // Attach Interrupt on RISING edge of flow sensor pulse
  attachInterrupt(digitalPinToInterrupt(FLOW_SENSOR_PIN), pulseCounterISR, RISING);

  previousMillis = millis();
}

void loop() {
  unsigned long currentMillis = millis();

  // Every 1 second, calculate flow rate and read tilt status
  if (currentMillis - previousMillis >= INTERVAL) {
    unsigned long duration = currentMillis - previousMillis;
    previousMillis = currentMillis;

    // --- Read Pulse Count Atomically ---
    noInterrupts();
    unsigned long currentPulses = pulseCount;
    pulseCount = 0;
    interrupts();

    // --- Calculate Flow Rate ---
    // flowRate = (Pulses / interval_in_seconds) / Calibration_Factor
    flowRate = ((1000.0 / duration) * currentPulses) / CALIBRATION_FACTOR;

    // Convert flow rate (L/min) to volume passed in this 1-second slice
    flowMilliLitres = (flowRate / 60.0) * 1000.0;
    totalMilliLitres += flowMilliLitres;

    // --- Read Tilt Sensor ---
    // Depending on sensor module: LOW = Normal / Tilted, HIGH = Opposite
    int tiltState = digitalRead(TILT_SENSOR_PIN);
    bool isTilted = (tiltState == HIGH); // Adjust if your module is active-LOW

    // Alert LED if tilted or flow is active
    digitalWrite(LED_PIN, isTilted ? HIGH : LOW);

    // --- Output to Serial Monitor ---
    Serial.println("========================================");
    Serial.printf("Flow Rate       : %.2f L/min\n", flowRate);
    Serial.printf("Current Vol (1s): %d mL\n", flowMilliLitres);
    Serial.printf("Total Volume    : %.3f L\n", totalMilliLitres / 1000.0);
    Serial.printf("Tilt Status     : %s\n", isTilted ? "⚠️ TILTED / DISPLACED" : "✅ STABLE");
    
    if (isTilted && flowRate > 0.1) {
      Serial.println("🚨 WARNING: Flow detected while system is tilted / leaking!");
    }
  }
}
```

---

### 3. How the Calculations Work

1. **Water Flow Rate ($Q$)**:
   $$\text{Flow Rate } (\text{L/min}) = \frac{\text{Pulse Frequency } (\text{Hz})}{7.5}$$
   * The YF-S201 sensor generates approximately **$450\text{ pulses}$ per Liter** of water.
   * If pulses counted in $1\,\text{second} = 15$, then $\text{Frequency} = 15\,\text{Hz} \implies Q = \frac{15}{7.5} = 2.0\,\text{L/min}$.

2. **Tilt Detection**:
   * The SW-520D sensor contains a metal ball that closes or opens the internal circuit depending on its angle of inclination ($> 15^\circ - 45^\circ$).
   * Useful for **tamper detection**, **container overturning alarms**, or **pipe displacement warnings**.

---

### 4. Next Steps & Enhancements

- **Calibrate the Flow Sensor**: Pour a known volume (e.g., precisely 1 Liter) into a measuring container and tune `CALIBRATION_FACTOR` until `totalMilliLitres` reads $1000\,\text{mL}$.
- **IoT Integration**: Send flow and tilt telemetry via **MQTT / Wi-Fi** to Home Assistant, ThingSpeak, or Blynk.
- **Automatic Shut-off**: Add a 12V Solenoid Valve via a Relay module to cut off water flow automatically if a tilt/leak event is triggered.

---

### 👤 Alwin T R (`2026-09-10T09:33:37Z`)

with lcd

---

### 🤖 AI Engineering Assistant (`2026-09-10T09:33:37Z`)

Here is the updated circuit and complete Arduino code incorporating an **I2C 16x2 LCD Display** alongside the water flow and tilt sensors.

---

### 1. Updated Wiring Diagram

ESP32 uses hardware I2C on **GPIO 21 (SDA)** and **GPIO 22 (SCL)** by default.

```
       +--------------------------------------------------------------------+
       |                              ESP32                                 |
       |                                                                    |
       |  VIN (5V)   GND     GPIO 21 (SDA)  GPIO 22 (SCL)  GPIO 27  GPIO 26 |
       +----+---------+-----------+--------------+-----------+--------+-----+
            |         |           |              |           |        |
            |         |           |              |           |        |
     +------+---------+-----------+--------------+---+       |        |
     | VCC     GND         SDA           SCL         |       |        |
     |                                               |       |        |
     |         16x2 I2C LCD Display (0x27)           |       |        |
     +-----------------------------------------------+       |        |
            |         |                                      |        |
            |         +-----------------------+              |        |
            |                                 |              |        |
     +------+---------+-----------------------+              |        |
     | VCC(Red)   GND(Black)   Signal(Yellow) |<-------------+        |
     |           Water Flow Sensor (YF-S201)  |                       |
     +----------------------------------------+                       |
            |         |                                               |
            |         +--------------------------------+              |
            |                                          |              |
     +------+---------+--------------------------------+--------------+
     | VCC(3.3V)     GND                            DO (Signal)       |
     |                 Tilt Sensor (SW-520D)                          |
     +----------------------------------------------------------------+
```

#### Pinout Summary

| Component | Pin | ESP32 Pin | Note |
| :--- | :--- | :--- | :--- |
| **I2C 16x2 LCD** | `VCC` | `VIN` (5V) | Power supply for backlight & display |
| | `GND` | `GND` | Common Ground |
| | `SDA` | `GPIO 21` | I2C Data |
| | `SCL` | `GPIO 22` | I2C Clock |
| **Water Flow (YF-S201)** | `Signal` | `GPIO 27` | Pulse interrupt pin |
| **Tilt Sensor (SW-520D)** | `DO` | `GPIO 26` | Digital input |

---

### 2. Required Library

In the Arduino IDE / PlatformIO, install:
* **`LiquidCrystal I2C`** by *Frank de Brabander* (or *Marco Schwartz*)

---

### 3. Complete Arduino C++ Code

This code uses formatted 16-character row buffers instead of `lcd.clear()` in the loop to eliminate screen flicker.

```cpp
#include <Arduino.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// ---------------- Pin Definitions ----------------
const int FLOW_SENSOR_PIN = 27;  // Flow sensor interrupt pin
const int TILT_SENSOR_PIN = 26;  // Tilt sensor digital pin
const int LED_PIN         = 2;   // Status LED

// ---------------- I2C LCD Settings ----------------
// Default address is usually 0x27 (or 0x3F for some displays)
LiquidCrystal_I2C lcd(0x27, 16, 2);

// ---------------- Flow Sensor Constants ----------------
// For YF-S201: Pulse frequency (Hz) = 7.5 * Q (L/min)
const float CALIBRATION_FACTOR = 7.5;

// ---------------- Global Variables ----------------
volatile unsigned long pulseCount = 0;

float flowRate = 0.0;               // L/min
unsigned int flowMilliLitres = 0;   // mL in the last second
unsigned long totalMilliLitres = 0; // Total accumulated mL

unsigned long previousMillis = 0;
const unsigned long INTERVAL = 1000; // 1-second update rate

// ---------------- Interrupt Service Routine ----------------
void IRAM_ATTR pulseCounterISR() {
  pulseCount++;
}

void setup() {
  Serial.begin(115200);

  // Initialize LCD
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("ESP32 Flow Meter");
  lcd.setCursor(0, 1);
  lcd.print("Initializing...");

  // Configure I/O Pins
  pinMode(LED_PIN, OUTPUT);
  pinMode(TILT_SENSOR_PIN, INPUT_PULLUP);
  pinMode(FLOW_SENSOR_PIN, INPUT_PULLUP);

  // Attach Interrupt to Flow Sensor
  attachInterrupt(digitalPinToInterrupt(FLOW_SENSOR_PIN), pulseCounterISR, RISING);

  delay(1500);
  lcd.clear();
  previousMillis = millis();
}

void loop() {
  unsigned long currentMillis = millis();

  // Execute update every 1 second
  if (currentMillis - previousMillis >= INTERVAL) {
    unsigned long duration = currentMillis - previousMillis;
    previousMillis = currentMillis;

    // --- Read Pulse Count Atomically ---
    noInterrupts();
    unsigned long currentPulses = pulseCount;
    pulseCount = 0;
    interrupts();

    // --- Calculate Flow Rates ---
    flowRate = ((1000.0 / duration) * currentPulses) / CALIBRATION_FACTOR;
    flowMilliLitres = (flowRate / 60.0) * 1000.0;
    totalMilliLitres += flowMilliLitres;

    // --- Read Tilt Sensor ---
    int tiltState = digitalRead(TILT_SENSOR_PIN);
    bool isTilted = (tiltState == HIGH); // Adjust logic if module is active-LOW

    // Alert LED
    digitalWrite(LED_PIN, isTilted ? HIGH : LOW);

    // --- Format Display Lines (16 Characters Wide) ---
    char line1[17];
    char line2[17];

    // Line 1: Flow rate & Tilt indicator
    // Example: "F: 2.3L/m [TILT]" or "F: 2.3L/m   [OK]"
    snprintf(line1, sizeof(line1), "F:%4.1fL/m %s", 
             flowRate, 
             isTilted ? "[TILT]" : "  [OK]");

    // Line 2: Total Volume in Liters
    // Example: "Tot:   12.450 L "
    float totalLiters = totalMilliLitres / 1000.0;
    snprintf(line2, sizeof(line2), "Tot:%8.3f L ", totalLiters);

    // --- Update LCD Display ---
    lcd.setCursor(0, 0);
    lcd.print(line1);
    lcd.setCursor(0, 1);
    lcd.print(line2);

    // --- Serial Output for Debugging ---
    Serial.printf("Flow: %.2f L/min | Total: %.3f L | Status: %s\n",
                  flowRate, totalLiters, isTilted ? "TILTED!" : "OK");
  }
}
```

---

### 4. LCD Display Output Preview

* **Normal Operation**:
  ```text
  +----------------+
  |F: 3.5L/m   [OK]|
  |Tot:   1.450 L  |
  +----------------+
  ```

* **When Tilted / Alert**:
  ```text
  +----------------+
  |F: 0.0L/m [TILT]|
  |Tot:   1.450 L  |
  +----------------+
  ```

---

### 5. Troubleshooting Tips

1. **Blank or Solid Blue Screen**:
   * Turn the small blue potentiometer on the back of the I2C module using a screwdriver to adjust the **contrast**.
2. **I2C Address Mismatch**:
   * If the screen does not respond, change `0x27` to `0x3F` in `LiquidCrystal_I2C lcd(0x3F, 16, 2);`.

---

### 👤 Alwin T R (`2026-09-10T09:34:44Z`)

when vibration sense lcd show earthquake detected

---

### 🤖 AI Engineering Assistant (`2026-09-10T09:34:45Z`)

Here is the updated code. It features **earthquake/vibration detection** that immediately interrupts normal monitoring, displays a prominent **"EARTHQUAKE DETECTED"** alert on the LCD, and flashes the alert LED/Buzzer.

Because earthquake tremors and vibration sensor pulses (SW-420 or SW-520D) happen in rapid millisecond bursts, this code uses an **interrupt with an auto-latching timer (3 seconds)** so brief tremors are never missed and the alert stays visible on the screen.

---

### 1. Wiring Diagram

```
       +--------------------------------------------------------------------+
       |                              ESP32                                 |
       |                                                                    |
       |  VIN (5V)   GND     GPIO 21 (SDA)  GPIO 22 (SCL)  GPIO 27  GPIO 26 |
       +----+---------+-----------+--------------+-----------+--------+-----+
            |         |           |              |           |        |
            |         |           |              |           |        |
     +------+---------+-----------+--------------+---+       |        |
     | VCC     GND         SDA           SCL         |       |        |
     |          16x2 I2C LCD Display (0x27)          |       |        |
     +-----------------------------------------------+       |        |
            |         |                                      |        |
     +------+---------+--------------------------------------+        |
     | VCC(Red)   GND(Black)   Signal(Yellow)                         |
     |           Water Flow Sensor (YF-S201)                          |
     +----------------------------------------------------------------+
            |         |                                               |
     +------+---------+-----------------------------------------------+
     | VCC(3.3V)     GND                            DO (Signal)       
     |       Vibration / Tilt Sensor (SW-420 / SW-520D)               
     +----------------------------------------------------------------+
```

---

### 2. Complete Arduino C++ Code

```cpp
#include <Arduino.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// ---------------- Pin Definitions ----------------
const int FLOW_SENSOR_PIN      = 27;  // Flow sensor interrupt pin
const int VIBRATION_SENSOR_PIN = 26;  // Vibration/Tilt interrupt pin
const int ALERT_LED_PIN        = 2;   // Onboard LED (or buzzer)

// ---------------- I2C LCD Settings ----------------
LiquidCrystal_I2C lcd(0x27, 16, 2);

// ---------------- Flow Sensor Constants ----------------
const float CALIBRATION_FACTOR = 7.5; // For YF-S201: F(Hz) = 7.5 * Q(L/min)

// ---------------- Global Variables ----------------
volatile unsigned long flowPulseCount = 0;
volatile bool earthquakeTriggered = false;
volatile unsigned long lastVibrationTime = 0;

const unsigned long ALERT_HOLD_MS = 3000; // Keep alert on screen for 3 seconds after tremor

float flowRate = 0.0;               // L/min
unsigned int flowMilliLitres = 0;   // mL in this second
unsigned long totalMilliLitres = 0; // Cumulative mL

unsigned long previousMillis = 0;
const unsigned long INTERVAL = 1000; // 1-second display update

// ---------------- Interrupt Service Routines ----------------
// 1. Water Flow ISR
void IRAM_ATTR flowPulseISR() {
  flowPulseCount++;
}

// 2. Vibration/Earthquake ISR (Triggers instantly on any vibration pulse)
void IRAM_ATTR vibrationISR() {
  earthquakeTriggered = true;
  lastVibrationTime = millis();
}

void setup() {
  Serial.begin(115200);

  // Initialize LCD
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("EARTHQUAKE &");
  lcd.setCursor(0, 1);
  lcd.print("FLOW MONITOR");

  // Configure Pins
  pinMode(ALERT_LED_PIN, OUTPUT);
  pinMode(FLOW_SENSOR_PIN, INPUT_PULLUP);
  pinMode(VIBRATION_SENSOR_PIN, INPUT_PULLUP);

  // Attach Hardware Interrupts
  attachInterrupt(digitalPinToInterrupt(FLOW_SENSOR_PIN), flowPulseISR, RISING);
  attachInterrupt(digitalPinToInterrupt(VIBRATION_SENSOR_PIN), vibrationISR, CHANGE);

  delay(2000);
  lcd.clear();
  previousMillis = millis();
}

void loop() {
  unsigned long currentMillis = millis();

  // Check if earthquake alert is currently active (holds for 3s after last shake)
  bool isAlertActive = earthquakeTriggered && (currentMillis - lastVibrationTime < ALERT_HOLD_MS);

  if (!isAlertActive) {
    earthquakeTriggered = false; // Reset trigger once time expires
    digitalWrite(ALERT_LED_PIN, LOW);
  } else {
    // Rapid flash LED during earthquake
    digitalWrite(ALERT_LED_PIN, (currentMillis / 100) % 2);
  }

  // --- Update Flow & Display every 1 second ---
  if (currentMillis - previousMillis >= INTERVAL) {
    unsigned long duration = currentMillis - previousMillis;
    previousMillis = currentMillis;

    // --- Read Pulse Count Atomically ---
    noInterrupts();
    unsigned long currentPulses = flowPulseCount;
    flowPulseCount = 0;
    interrupts();

    // --- Flow Calculations ---
    flowRate = ((1000.0 / duration) * currentPulses) / CALIBRATION_FACTOR;
    flowMilliLitres = (flowRate / 60.0) * 1000.0;
    totalMilliLitres += flowMilliLitres;
    float totalLiters = totalMilliLitres / 1000.0;

    // --- Screen Rendering ---
    if (isAlertActive) {
      // 🚨 EARTHQUAKE WARNING SCREEN
      lcd.setCursor(0, 0);
      lcd.print("** EARTHQUAKE **");
      lcd.setCursor(0, 1);
      lcd.print("   DETECTED!    ");

      Serial.println("🚨 ALERT: EARTHQUAKE / VIBRATION DETECTED!");
    } 
    else {
      // 💧 NORMAL FLOW MONITOR SCREEN
      char line1[17];
      char line2[17];

      snprintf(line1, sizeof(line1), "Flow: %4.1f L/min", flowRate);
      snprintf(line2, sizeof(line2), "Total:%7.2f L  ", totalLiters);

      lcd.setCursor(0, 0);
      lcd.print(line1);
      lcd.setCursor(0, 1);
      lcd.print(line2);

      Serial.printf("Flow: %.2f L/min | Total: %.3f L | Status: NORMAL\n", flowRate, totalLiters);
    }
  }
}
```

---

### 3. Screen States

* **Normal Monitoring**:
  ```text
  +----------------+
  |Flow:  3.5 L/min|
  |Total:   1.45 L |
  +----------------+
  ```

* **When Vibration / Tremor is Sensed**:
  ```text
  +----------------+
  |** EARTHQUAKE **|
  |   DETECTED!    |
  +----------------+
  ```

---

### 4. Key Highlights

1. **Zero Missed Tremors**: Since seismic vibrations happen in milliseconds, the `vibrationISR()` interrupt catches the vibration immediately, even if the main program is calculating flow rates.
2. **Alert Latch (3 seconds)**: The alert stays on the screen for at least 3 seconds after the vibration stops, preventing the screen from flickering on and off.
3. **Continuous Background Metering**: Water flow pulses are still counted safely in the background during an earthquake alert.

---

### 👤 Alwin T R (`2026-09-10T09:35:46Z`)

water sensor give aler as earhquake

---

### 🤖 AI Engineering Assistant (`2026-09-10T09:35:47Z`)

This happens for two main reasons:

1. **Mechanical Vibration from the Turbine**: When water flows through the sensor, the internal turbine wheel spins rapidly. If the vibration/tilt sensor is on the same board or pipe, it picks up this mechanical spinning and mistakes it for an earthquake.
2. **Over-Sensitive Interrupt**: A single electrical tick or tiny shake immediately triggered the earthquake alert. Real earthquakes produce **sustained shaking (multiple pulses in a short window)**, not just 1 pulse.

---

### Solution: Earthquake Filtering Code

This updated code uses an **Earthquake Validation Filter**:
- It counts the number of vibration pulses in a **500 ms window**.
- **Small vibrations/water flow (< 5 pulses)** $\rightarrow$ **Ignored** as normal water flow noise.
- **Sustained shaking ($\ge 5$ pulses in 500 ms)** $\rightarrow$ **Earthquake Alert Triggered**.

```cpp
#include <Arduino.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// ---------------- Pin Definitions ----------------
const int FLOW_SENSOR_PIN      = 27;  // Water flow sensor
const int VIBRATION_SENSOR_PIN = 26;  // Vibration/Tilt sensor
const int ALERT_LED_PIN        = 2;   // Status LED / Buzzer

// ---------------- I2C LCD Settings ----------------
LiquidCrystal_I2C lcd(0x27, 16, 2);

// ---------------- Flow Sensor Constants ----------------
const float CALIBRATION_FACTOR = 7.5; // YF-S201

// ---------------- Earthquake Filter Settings ----------------
// Minimum vibration pulses in a 500ms window to classify as an EARTHQUAKE
const int VIBRATION_THRESHOLD = 5;    // Adjust this (3 to 10) for your sensor
const unsigned long VIBRATION_WINDOW_MS = 500;
const unsigned long ALERT_HOLD_MS = 3000; // Hold alert on screen for 3 seconds

// ---------------- Volatile ISR Variables ----------------
volatile unsigned long flowPulseCount = 0;
volatile unsigned int vibrationPulseCount = 0;
volatile unsigned long lastVibrationTime = 0;

// ---------------- Global State Variables ----------------
bool earthquakeAlertActive = false;
unsigned long alertStartTime = 0;

float flowRate = 0.0;
unsigned int flowMilliLitres = 0;
unsigned long totalMilliLitres = 0;

unsigned long previousMillis = 0;
const unsigned long INTERVAL = 1000;

// ---------------- Interrupt Service Routines ----------------
void IRAM_ATTR flowPulseISR() {
  flowPulseCount++;
}

void IRAM_ATTR vibrationISR() {
  vibrationPulseCount++;
  lastVibrationTime = millis();
}

void setup() {
  Serial.begin(115200);

  // Initialize LCD
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("ESP32 Monitor");
  lcd.setCursor(0, 1);
  lcd.print("Calibrating...");

  pinMode(ALERT_LED_PIN, OUTPUT);
  pinMode(FLOW_SENSOR_PIN, INPUT_PULLUP);
  pinMode(VIBRATION_SENSOR_PIN, INPUT_PULLUP);

  // Attach Interrupts
  attachInterrupt(digitalPinToInterrupt(FLOW_SENSOR_PIN), flowPulseISR, RISING);
  attachInterrupt(digitalPinToInterrupt(VIBRATION_SENSOR_PIN), vibrationISR, FALLING);

  delay(1500);
  lcd.clear();
  previousMillis = millis();
}

void loop() {
  unsigned long currentMillis = millis();

  // ---------------- 1. EARTHQUAKE VALIDATION LOGIC ----------------
  // Check vibration pulses received in the recent window
  if (vibrationPulseCount >= VIBRATION_THRESHOLD) {
    earthquakeAlertActive = true;
    alertStartTime = currentMillis;
    vibrationPulseCount = 0; // Reset counter
  }

  // Reset the pulse window if no new shakes arrived
  if (currentMillis - lastVibrationTime > VIBRATION_WINDOW_MS) {
    vibrationPulseCount = 0;
  }

  // Auto-clear alert after 3 seconds of calm
  if (earthquakeAlertActive && (currentMillis - alertStartTime > ALERT_HOLD_MS)) {
    earthquakeAlertActive = false;
  }

  // LED Alert feedback
  if (earthquakeAlertActive) {
    digitalWrite(ALERT_LED_PIN, (currentMillis / 100) % 2); // Fast Blink
  } else {
    digitalWrite(ALERT_LED_PIN, LOW);
  }

  // ---------------- 2. DISPLAY & FLOW UPDATE (EVERY 1 SEC) ----------------
  if (currentMillis - previousMillis >= INTERVAL) {
    unsigned long duration = currentMillis - previousMillis;
    previousMillis = currentMillis;

    // Read Flow Sensor Pulses
    noInterrupts();
    unsigned long currentFlowPulses = flowPulseCount;
    flowPulseCount = 0;
    interrupts();

    // Calculate Flow
    flowRate = ((1000.0 / duration) * currentFlowPulses) / CALIBRATION_FACTOR;
    flowMilliLitres = (flowRate / 60.0) * 1000.0;
    totalMilliLitres += flowMilliLitres;
    float totalLiters = totalMilliLitres / 1000.0;

    // Render Display
    if (earthquakeAlertActive) {
      lcd.setCursor(0, 0);
      lcd.print("** EARTHQUAKE **");
      lcd.setCursor(0, 1);
      lcd.print("   DETECTED!    ");
      Serial.println("🚨 [ALERT] Sustained Earthquake Shaking Detected!");
    } else {
      char line1[17];
      char line2[17];

      snprintf(line1, sizeof(line1), "Flow: %4.1f L/min", flowRate);
      snprintf(line2, sizeof(line2), "Total:%7.2f L  ", totalLiters);

      lcd.setCursor(0, 0);
      lcd.print(line1);
      lcd.setCursor(0, 1);
      lcd.print(line2);

      Serial.printf("Flow: %.2f L/min | Total: %.3f L | Status: OK\n", flowRate, totalLiters);
    }
  }
}
```

---

### Hardware Adjustments to Stop False Alarms

1. **Adjust the Blue Trimpot on the Vibration Sensor**:
   * Turn the small blue potentiometer screw **counter-clockwise** to lower its sensitivity so it ignores the hum of flowing water.
2. **Physical Separation**:
   * Do **not** tape or mount the vibration sensor directly onto the water pipe or the water flow meter. Mount it firmly to a **wall, floor, or chassis frame** instead.

---

### 👤 Alwin T R (`2026-09-10T09:37:05Z`)

when give fllood alert and buzzer alert

---

### 🤖 AI Engineering Assistant (`2026-09-10T09:37:05Z`)

Here is the complete updated system that includes **Flood Alert** (triggered when water flow exceeds a dangerous threshold or pipe bursts) and an **Active Buzzer** with distinct audible alarm patterns for both **Earthquakes** and **Floods**.

---

### 1. Circuit & Wiring Diagram

```
       +---------------------------------------------------------------------------------+
       |                                     ESP32                                       |
       |                                                                                 |
       |  VIN (5V)   GND     GPIO 21 (SDA)  GPIO 22 (SCL)  GPIO 27  GPIO 26  GPIO 25  GPIO 2 |
       +----+---------+-----------+--------------+-----------+--------+--------+--------+----+
            |         |           |              |           |        |        |        |
     +------+---------+-----------+--------------+---+       |        |        |        |
     | VCC     GND         SDA           SCL         |       |        |        |        |
     |          16x2 I2C LCD Display (0x27)          |       |        |        |        |
     +-----------------------------------------------+       |        |        |        |
            |         |                                      |        |        |        |
     +------+---------+--------------------------------------+        |        |        |
     | VCC(Red)   GND(Black)   Signal(Yellow)                         |        |        |
     |           Water Flow Sensor (YF-S201)                          |        |        |
     +----------------------------------------------------------------+        |        |
            |         |                                                        |        |
     +------+---------+--------------------------------------------------------+        |
     | VCC(3.3V)     GND                            DO (Signal)                         |
     |       Vibration / Tilt Sensor (SW-420 / SW-520D)                                 |
     +----------------------------------------------------------------------------------+
            |         |                                                                 |
     +------+---------+-----------------------------------------------------------------+
     |                 GND (-)                                           VCC / Signal (+)
     |                                 5V Active Buzzer Module
     +-----------------------------------------------------------------------------------
```

---

### 2. Alert Conditions & Buzzer Tones

| Event | Trigger Condition | LCD Display | Buzzer Alarm Pattern |
| :--- | :--- | :--- | :--- |
| **Earthquake** | $\ge 5$ vibration pulses in $500\,\text{ms}$ | `** EARTHQUAKE **`<br>`   DETECTED!    ` | Rapid Staccato Beep (50ms ON / 50ms OFF) |
| **Flood** | Flow Rate $\ge 10.0\,\text{L/min}$ *(Configurable)* | `** FLOOD ALERT **`<br>` HIGH WATER FLOW ` | Slow Warning Pulse (250ms ON / 250ms OFF) |
| **Dual Emergency** | Earthquake **and** High Flow simultaneously | `! DUAL DANGER ! `<br>`QUAKE + FLOODING ` | Continuous Siren |
| **Normal** | Safe flow rate & stable ground | `Flow:  X.X L/min`<br>`Total: X.XX L   ` | **Silent (OFF)** |

---

### 3. Complete Arduino C++ Code

```cpp
#include <Arduino.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// ---------------- Pin Definitions ----------------
const int FLOW_SENSOR_PIN      = 27;  // Water flow sensor
const int VIBRATION_SENSOR_PIN = 26;  // Vibration/Tilt sensor
const int BUZZER_PIN           = 25;  // Active Buzzer (+)
const int ALERT_LED_PIN        = 2;   // Onboard status LED

// ---------------- I2C LCD Settings ----------------
LiquidCrystal_I2C lcd(0x27, 16, 2);

// ---------------- Threshold Settings ----------------
const float CALIBRATION_FACTOR   = 7.5;   // YF-S201 sensor factor
const float FLOOD_FLOW_THRESHOLD = 10.0;  // Liters/min threshold for Flood Alert
const int   VIBRATION_THRESHOLD  = 5;     // Pulses within 500ms for Earthquake
const unsigned long VIBRATION_WINDOW_MS = 500;
const unsigned long ALERT_HOLD_MS       = 3000; // Hold alert for 3 seconds

// ---------------- Volatile ISR Variables ----------------
volatile unsigned long flowPulseCount = 0;
volatile unsigned int vibrationPulseCount = 0;
volatile unsigned long lastVibrationTime = 0;

// ---------------- Global State Variables ----------------
bool earthquakeActive = false;
bool floodActive      = false;
unsigned long earthquakeAlertStartTime = 0;

float flowRate = 0.0;
unsigned int flowMilliLitres = 0;
unsigned long totalMilliLitres = 0;

unsigned long previousMillis = 0;
const unsigned long INTERVAL = 1000; // 1-second display rate

// ---------------- Interrupt Service Routines ----------------
void IRAM_ATTR flowPulseISR() {
  flowPulseCount++;
}

void IRAM_ATTR vibrationISR() {
  vibrationPulseCount++;
  lastVibrationTime = millis();
}

void setup() {
  Serial.begin(115200);

  // Initialize LCD
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("DISASTER MONITOR");
  lcd.setCursor(0, 1);
  lcd.print("SYSTEM READY...");

  // Configure Pins
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(ALERT_LED_PIN, OUTPUT);
  pinMode(FLOW_SENSOR_PIN, INPUT_PULLUP);
  pinMode(VIBRATION_SENSOR_PIN, INPUT_PULLUP);

  // Initial outputs LOW
  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(ALERT_LED_PIN, LOW);

  // Attach Interrupts
  attachInterrupt(digitalPinToInterrupt(FLOW_SENSOR_PIN), flowPulseISR, RISING);
  attachInterrupt(digitalPinToInterrupt(VIBRATION_SENSOR_PIN), vibrationISR, FALLING);

  delay(2000);
  lcd.clear();
  previousMillis = millis();
}

// ---------------- Non-Blocking Buzzer Controller ----------------
void handleAlarms(unsigned long currentMillis) {
  if (earthquakeActive && floodActive) {
    // Dual Danger: Continuous Siren & Fast Flash
    digitalWrite(BUZZER_PIN, HIGH);
    digitalWrite(ALERT_LED_PIN, (currentMillis / 100) % 2);
  } 
  else if (earthquakeActive) {
    // Earthquake: Fast Staccato Beep (50ms ON / 50ms OFF)
    bool beepState = (currentMillis % 100) < 50;
    digitalWrite(BUZZER_PIN, beepState ? HIGH : LOW);
    digitalWrite(ALERT_LED_PIN, beepState ? HIGH : LOW);
  } 
  else if (floodActive) {
    // Flood: Slow Pulsing Tone (250ms ON / 250ms OFF)
    bool beepState = (currentMillis % 500) < 250;
    digitalWrite(BUZZER_PIN, beepState ? HIGH : LOW);
    digitalWrite(ALERT_LED_PIN, beepState ? HIGH : LOW);
  } 
  else {
    // Safe / Normal: Silence
    digitalWrite(BUZZER_PIN, LOW);
    digitalWrite(ALERT_LED_PIN, LOW);
  }
}

void loop() {
  unsigned long currentMillis = millis();

  // 1. Check Earthquake Trigger
  if (vibrationPulseCount >= VIBRATION_THRESHOLD) {
    earthquakeActive = true;
    earthquakeAlertStartTime = currentMillis;
    vibrationPulseCount = 0;
  }
  if (currentMillis - lastVibrationTime > VIBRATION_WINDOW_MS) {
    vibrationPulseCount = 0;
  }
  if (earthquakeActive && (currentMillis - earthquakeAlertStartTime > ALERT_HOLD_MS)) {
    earthquakeActive = false;
  }

  // 2. Play Buzzer & LED alarm patterns
  handleAlarms(currentMillis);

  // 3. Flow Calculations & LCD Update (Every 1 Second)
  if (currentMillis - previousMillis >= INTERVAL) {
    unsigned long duration = currentMillis - previousMillis;
    previousMillis = currentMillis;

    // Read Flow Pulses safely
    noInterrupts();
    unsigned long currentFlowPulses = flowPulseCount;
    flowPulseCount = 0;
    interrupts();

    // Calculate Flow Rate and Volume
    flowRate = ((1000.0 / duration) * currentFlowPulses) / CALIBRATION_FACTOR;
    flowMilliLitres = (flowRate / 60.0) * 1000.0;
    totalMilliLitres += flowMilliLitres;
    float totalLiters = totalMilliLitres / 1000.0;

    // Check Flood Condition
    floodActive = (flowRate >= FLOOD_FLOW_THRESHOLD);

    // 4. Update LCD Display based on priority
    if (earthquakeActive && floodActive) {
      lcd.setCursor(0, 0);
      lcd.print("! DUAL DANGER ! ");
      lcd.setCursor(0, 1);
      lcd.print("QUAKE + FLOODING");
      Serial.println("🚨🚨 [CRITICAL] EARTHQUAKE & FLOOD DETECTED!");
    } 
    else if (earthquakeActive) {
      lcd.setCursor(0, 0);
      lcd.print("** EARTHQUAKE **");
      lcd.setCursor(0, 1);
      lcd.print("   DETECTED!    ");
      Serial.println("🚨 [ALERT] Earthquake / Tremor Active!");
    } 
    else if (floodActive) {
      lcd.setCursor(0, 0);
      lcd.print("** FLOOD ALERT **");
      lcd.setCursor(0, 1);
      lcd.print(" HIGH WATER FLOW");
      Serial.printf("🌊 [ALERT] Flood / Pipe Burst! Flow: %.2f L/min\n", flowRate);
    } 
    else {
      char line1[17];
      char line2[17];

      snprintf(line1, sizeof(line1), "Flow: %4.1f L/min", flowRate);
      snprintf(line2, sizeof(line2), "Total:%7.2f L  ", totalLiters);

      lcd.setCursor(0, 0);
      lcd.print(line1);
      lcd.setCursor(0, 1);
      lcd.print(line2);

      Serial.printf("Flow: %.2f L/min | Total: %.3f L | Status: NORMAL\n", flowRate, totalLiters);
    }
  }
}
```

---

### 4. Customizing the Thresholds

- **Change Flood Flow Limit**: Modify `FLOOD_FLOW_THRESHOLD = 10.0;` to a lower value (e.g. `5.0` L/min) for smaller domestic pipes or `20.0` L/min for main supply lines.
- **Change Earthquake Sensitivity**: Adjust `VIBRATION_THRESHOLD = 5;` up or down to calibrate for your specific sensor.

---

### 👤 Alwin T R (`2026-09-10T09:53:33Z`)

upload to esp32

---

### ⚡ Automated Actions (`2026-09-10T09:53:33Z`)

- **run_command**: Check COM ports and build tools

---

### ⚡ Automated Actions (`2026-09-10T09:53:40Z`)

- **run_command**: Search for Arduino and PlatformIO

---

### ⚡ Automated Actions (`2026-09-10T09:53:47Z`)

- **run_command**: Find arduino-cli binary

---

### ⚡ Automated Actions (`2026-09-10T09:53:57Z`)

- **manage_task**: Check task status

---

### ⚡ Automated Actions (`2026-09-10T09:53:59Z`)

- **manage_task**: Check task status

---

### 🤖 AI Engineering Assistant (`2026-09-10T09:54:02Z`)

I am checking your development environment and connected ESP32 boards to prepare the code compilation and upload.

---

### ⚡ Automated Actions (`2026-09-10T09:54:05Z`)

- **run_command**: List boards with arduino-cli

---

### 🤖 AI Engineering Assistant (`2026-09-10T09:54:13Z`)

I am querying `arduino-cli` to detect the exact ESP32 board model on COM11 and will proceed shortly.

---

### ⚡ Automated Actions (`2026-09-10T09:54:15Z`)

- **run_command**: List cores and libraries

---

### 🤖 AI Engineering Assistant (`2026-09-10T09:54:24Z`)

I am checking if the ESP32 core package and LiquidCrystal_I2C library are installed.

---

### ⚡ Automated Actions (`2026-09-10T09:54:27Z`)

- **write_to_file**: Create sketch file

---

### ⚡ Automated Actions (`2026-09-10T09:54:33Z`)

- **run_command**: Create sketch on disk

---

### ⚡ Automated Actions (`2026-09-10T09:54:40Z`)

- **run_command**: Write sketch to disk

---

### ⚡ Automated Actions (`2026-09-10T09:54:47Z`)

- **run_command**: Create directory

---

### ⚡ Automated Actions (`2026-09-10T09:54:51Z`)

- **run_command**: Write sketch file

---

### ⚡ Automated Actions (`2026-09-10T09:55:11Z`)

- **run_command**: Compile sketch with arduino-cli

---

### 🤖 AI Engineering Assistant (`2026-09-10T09:55:19Z`)

I am compiling the Arduino sketch for the ESP32 (FQBN: `esp32:esp32:esp32`) and will upload it immediately after compilation succeeds.

---

### ⚡ Automated Actions (`2026-09-10T09:55:41Z`)

- **run_command**: Write corrected sketch

---

### ⚡ Automated Actions (`2026-09-10T09:55:49Z`)

- **run_command**: Write sketch file cleanly

---

### ⚡ Automated Actions (`2026-09-10T09:55:55Z`)

- **run_command**: Test Python hex

---

### ⚡ Automated Actions (`2026-09-10T09:55:59Z`)

- **run_command**: Generate write_helper script

---

### ⚡ Automated Actions (`2026-09-10T09:56:06Z`)

- **run_command**: Write sketch to disk

---

### ⚡ Automated Actions (`2026-09-10T09:56:24Z`)

- **run_command**: Compile sketch with arduino-cli

---

### 🤖 AI Engineering Assistant (`2026-09-10T09:56:32Z`)

I am compiling the sketch for your ESP32 board and will proceed to upload once finished.

---

### ⚡ Automated Actions (`2026-09-10T09:56:46Z`)

- **run_command**: Write clean sketch to disk

---

### ⚡ Automated Actions (`2026-09-10T09:56:57Z`)

- **run_command**: Setup sketch writer

---

### ⚡ Automated Actions (`2026-09-10T09:57:05Z`)

- **manage_task**: Kill task 62

---

### ⚡ Automated Actions (`2026-09-10T09:57:09Z`)

- **write_to_file**: Write sketch file cleanly

---

### ⚡ Automated Actions (`2026-09-10T09:57:16Z`)

- **run_command**: Compile and upload to ESP32

---

### 🤖 AI Engineering Assistant (`2026-09-10T09:57:24Z`)

I have started compiling and uploading the sketch to your ESP32 on **`COM11`**. Please keep the board connected.

---

### 🤖 AI Engineering Assistant (`2026-09-10T09:58:01Z`)

### **Successfully Uploaded to ESP32!**

The program has been compiled and flashed directly to your ESP32 on **`COM11`**.

---

### **System Configuration & Pinout Reference**

| Component | Pin | ESP32 GPIO | Description |
| :--- | :--- | :--- | :--- |
| **I2C 16x2 LCD** | `SDA` | **GPIO 21** | I2C Data Line |
| | `SCL` | **GPIO 22** | I2C Clock Line |
| | `VCC` / `GND` | `5V (VIN)` / `GND` | Power & Ground |
| **Water Flow Sensor** | Signal (Yellow) | **GPIO 27** | Hardware Interrupt pulse counter (`RISING`) |
| **Vibration / Tilt** | DO (Digital Out) | **GPIO 26** | Hardware Interrupt tremor counter (`FALLING`) |
| **Active Buzzer** | (+) Signal | **GPIO 25** | Audible alarm output |
| **Status LED** | Onboard LED | **GPIO 2** | Visual flash indicator |

---

### **How the System Operates**

1. **Normal Flow Monitoring**:
   * **LCD Line 1**: `Flow:  X.X L/min`
   * **LCD Line 2**: `Total: X.XX L   `
   * **Buzzer / LED**: Silent / Off

2. **Earthquake / Tremor Detection**:
   * **Trigger**: Sustained shaking ($\ge 5$ vibration pulses within $500\,\text{ms}$, filtering out normal pipe turbulence)
   * **LCD**:
     ```text
     ** EARTHQUAKE **
        DETECTED!    
     ```
   * **Alarm**: Rapid staccato buzzer beeps (50ms ON / 50ms OFF) + fast blinking LED for 3 seconds.

3. **Flood / Pipe Burst Alert**:
   * **Trigger**: Flow rate $\ge 10.0\,\text{L/min}$
   * **LCD**:
     ```text
     ** FLOOD ALERT **
      HIGH WATER FLOW
     ```
   * **Alarm**: Slow pulsing buzzer tone (250ms ON / 250ms OFF).

4. **Dual Emergency**:
   * **Trigger**: Earthquake and High Flow triggered simultaneously.
   * **LCD**: `! DUAL DANGER !` / `QUAKE + FLOODING`
   * **Alarm**: Continuous siren.

---

### 👤 Alwin T R (`2026-09-10T09:58:45Z`)

lcd not working

---

### ⚡ Automated Actions (`2026-09-10T09:58:45Z`)

- **write_to_file**: Write I2C scanner sketch

---

### ⚡ Automated Actions (`2026-09-10T09:58:52Z`)

- **run_command**: Upload I2C scanner

---

### 🤖 AI Engineering Assistant (`2026-09-10T09:59:00Z`)

I am uploading an I2C diagnostic scanner to your ESP32 to test the LCD connections and detect its exact I2C address.

---

### ⚡ Automated Actions (`2026-09-10T09:59:38Z`)

- **run_command**: Read serial port output

---

### ⚡ Automated Actions (`2026-09-10T09:59:43Z`)

- **run_command**: Capture I2C scan result

---

### ⚡ Automated Actions (`2026-09-10T09:59:50Z`)

- **write_to_file**: Update scanner sketch

---

### ⚡ Automated Actions (`2026-09-10T09:59:53Z`)

- **run_command**: Compile and flash continuous scanner

---

### 🤖 AI Engineering Assistant (`2026-09-10T10:00:01Z`)

I am flashing the continuous I2C scanner to check the LCD status live.

---

### ⚡ Automated Actions (`2026-09-10T10:00:31Z`)

- **run_command**: Read live I2C logs

---

### ⚡ Automated Actions (`2026-09-10T10:00:36Z`)

- **run_command**: Read serial bytes

---

### ⚡ Automated Actions (`2026-09-10T10:00:48Z`)

- **run_command**: Read serial lines

---

### ⚡ Automated Actions (`2026-09-10T10:00:52Z`)

- **run_command**: Capture loop output

---

### ⚡ Automated Actions (`2026-09-10T10:00:59Z`)

- **write_to_file**: Update sketch file

---

### ⚡ Automated Actions (`2026-09-10T10:01:07Z`)

- **run_command**: Compile and upload updated sketch

---

### 🤖 AI Engineering Assistant (`2026-09-10T10:01:16Z`)

I am uploading the updated sketch with explicit I2C bus initialization to your ESP32.

---

### 🤖 AI Engineering Assistant (`2026-09-10T10:01:45Z`)

The I2C hardware scan confirmed that **your LCD is connected and responding at I2C address `0x27` on GPIO 21 & GPIO 22**. The updated sketch has just been flashed.

If the screen is currently lit but blank (or showing solid boxes), follow these 3 quick checks:

---

### 1. Adjust the Blue Contrast Screw *(Most Common)*
On the **back of your LCD screen** (on the small black I2C backpack board), there is a **small blue square potentiometer with a screw head**.
* Take a small flathead screwdriver and **slowly turn the screw** clockwise or counter-clockwise.
* You will see the text contrast fade in from invisible $\rightarrow$ clear sharp text $\rightarrow$ solid white blocks.
* Stop turning when the text `SYSTEM STARTING` / `Flow: 0.0 L/min` becomes clearly visible.

---

### 2. Check Power Supply (5V vs 3.3V)
* **VCC** of the I2C LCD **must be connected to 5V (VIN)** on the ESP32, not `3.3V`.
* 16x2 LCDs require 5V to drive the liquid crystal characters. If connected to 3.3V, the backlight might glow dimly, but characters will not render.

---

### 3. Check the Backlight Jumper Cap
* Look on the back of the LCD I2C module: there is a **2-pin header labeled "LED" with a small black jumper cap** bridging the two pins.
* Make sure this black jumper is firmly seated. (If removed, the backlight stays completely OFF).

---

### Pin Connection Check

| LCD Pin | ESP32 Pin |
| :--- | :--- |
| **GND** | `GND` |
| **VCC** | **`VIN` (5V)** |
| **SDA** | **`GPIO 21`** |
| **SCL** | **`GPIO 22`** |

---

### 👤 Alwin T R (`2026-09-10T10:01:54Z`)

backlight is low

---

### 🤖 AI Engineering Assistant (`2026-09-10T10:01:54Z`)

A dim or low backlight on an ESP32 occurs because the LCD is receiving **3.3V** instead of **5V**.

---

### **How to Fix It (Full Brightness)**

#### 1. Move `VCC` to the `VIN` (5V) Pin
* **Current Wiring (Too Dim):** LCD `VCC` $\rightarrow$ ESP32 `3.3V`
* **Fix (Full Brightness):** Move LCD `VCC` $\rightarrow$ **ESP32 `VIN` (or `5V` / `VBUS`)**

> **Why this happens:** The backlight LED on the 16x2 I2C module has a built-in resistor sized for **5 Volts**. When powered with 3.3V, the LED does not receive enough voltage or current to glow at full brightness.

---

#### 2. Check the Backlight Jumper on the Back
On the rear of the LCD's I2C board, find the **2-pin header labeled "LED"**:
* Ensure the **black plastic jumper cap** is firmly pushed down all the way over both pins.
* If the jumper is loose or making poor contact, the backlight will flicker or be dim.

---

### **Summary of LCD Connections**

| LCD Pin | Correct ESP32 Pin |
| :--- | :--- |
| **VCC** | **`VIN` (5V from USB)** *(Fixes Dim Backlight)* |
| **GND** | **`GND`** |
| **SDA** | **`GPIO 21`** |
| **SCL** | **`GPIO 22`** |

---

### 👤 Alwin T R (`2026-09-10T10:18:28Z`)

i need  only flood alert and earthquake detected in display

---

### ⚡ Automated Actions (`2026-09-10T10:18:28Z`)

- **write_to_file**: Update sketch for alert-only display

---

### ⚡ Automated Actions (`2026-09-10T10:18:37Z`)

- **run_command**: Compile and upload to ESP32

---

### 🤖 AI Engineering Assistant (`2026-09-10T10:18:46Z`)

I am compiling and uploading the updated sketch to your ESP32.

---

### 🤖 AI Engineering Assistant (`2026-09-10T10:19:21Z`)

### **Updated & Uploaded to ESP32!**

The LCD is now configured to **only show clean disaster alert messages and safe status**:

---

### **LCD Display Screen States**

#### 1. **Normal State (Safe)**
```text
+----------------+
|  STATUS: SAFE  |
| MONITORING...  |
+----------------+
```
*Buzzer & LED:* **Off**

---

#### 2. **When Earthquake / Vibration is Detected**
```text
+----------------+
|** EARTHQUAKE **|
|   DETECTED!    |
+----------------+
```
*Buzzer & LED:* **Rapid staccato beeps & flashes** for 3 seconds.

---

#### 3. **When Flood / High Water Flow is Detected**
```text
+----------------+
|** FLOOD ALERT **|
| HIGH WATER FLOW|
+----------------+
```
*Buzzer & LED:* **Slow pulsing alarm** as long as high flow continues.

---

#### 4. **When Both Happen (Dual Danger)**
```text
+----------------+
|! DUAL DANGER ! |
|QUAKE + FLOODING|
+----------------+
```
*Buzzer & LED:* **Continuous alarm siren.**

---

### 👤 Alwin T R (`2026-09-10T10:22:29Z`)

not show flood detected

---

### ⚡ Automated Actions (`2026-09-10T10:22:29Z`)

- **write_to_file**: Update sketch for flood detection

---

### ⚡ Automated Actions (`2026-09-10T10:22:37Z`)

- **run_command**: Compile and upload to ESP32

---

### 🤖 AI Engineering Assistant (`2026-09-10T10:22:46Z`)

I am uploading the updated sketch with fast flow detection and a 3-second display latch for the Flood Alert.

---

### ⚡ Automated Actions (`2026-09-10T10:23:12Z`)

- **run_command**: Upload to ESP32 on COM11

---

### 🤖 AI Engineering Assistant (`2026-09-10T10:23:20Z`)

I closed any background serial locks on `COM11` and retried the upload.

---

### ⚡ Automated Actions (`2026-09-10T10:23:38Z`)

- **run_command**: Check processes holding serial port

---

### 🤖 AI Engineering Assistant (`2026-09-10T10:23:47Z`)

Checking open processes that may be using `COM11`.

---

### ⚡ Automated Actions (`2026-09-10T10:24:01Z`)

- **run_command**: List Arduino processes

---

### ⚡ Automated Actions (`2026-09-10T10:24:05Z`)

- **run_command**: Kill background serial locker

---

### ⚡ Automated Actions (`2026-09-10T10:24:08Z`)

- **run_command**: Compile and upload sketch

---

### 🤖 AI Engineering Assistant (`2026-09-10T10:24:17Z`)

I have freed the serial port and uploaded the new sketch to your ESP32.

---

### 🤖 AI Engineering Assistant (`2026-09-10T10:24:40Z`)

### **Uploaded Successfully to ESP32!**

The flood detection sensitivity and response speed have been updated:

---

### **What Changed:**
1. **Immediate Trigger on Flow**: The flow threshold was lowered (`0.5 L/min` / any pulse detected) so that even small trickles, pouring water, or blowing air into the turbine will instantly trigger the alert.
2. **3-Second Screen Hold**: When water flow is detected, the **`** FLOOD ALERT **` / `   DETECTED!    `** message will remain clearly visible on the LCD for at least 3 seconds.
3. **Fast Refresh (200ms)**: The ESP32 now refreshes the flow state every 200ms for immediate on-screen response.

---

### **Testing Your Sensors Now:**

* **Test Flood**: Blow gently into the water flow sensor or pass water through it $\rightarrow$ The LCD will instantly switch to:
  ```text
  +------------------+
  |** FLOOD ALERT ** |
  |   DETECTED!      |
  +------------------+
  ```
  *(Buzzer will beep with slow pulsing alarm)*

* **Test Earthquake**: Shake or tap the vibration sensor $\rightarrow$ The LCD will switch to:
  ```text
  +------------------+
  |** EARTHQUAKE **  |
  |   DETECTED!      |
  +------------------+
  ```
  *(Buzzer will beep with rapid staccato beeps)*

* **When Idle / Normal**:
  ```text
  +------------------+
  |  STATUS: SAFE    |
  | MONITORING...    |
  +------------------+
  ```

---

### 👤 Alwin T R (`2026-09-10T10:25:49Z`)

vibration sensor change the value to high.to avoid detect suddenly.give dynamic solution for it

---

### ⚡ Automated Actions (`2026-09-10T10:25:49Z`)

- **write_to_file**: Update sketch with dynamic filter

---

### ⚡ Automated Actions (`2026-09-10T10:25:58Z`)

- **run_command**: Compile and upload sketch

---

### 🤖 AI Engineering Assistant (`2026-09-10T10:26:06Z`)

I am compiling and uploading the dynamic seismic filter sketch to your ESP32.

---

### 🤖 AI Engineering Assistant (`2026-09-10T10:26:38Z`)

### **Dynamic Seismic Filter Uploaded to ESP32!**

To eliminate sudden false alarms from accidental bumps, table knocks, or water pipe vibrations, this update implements a **Dynamic Leaky Bucket (Seismic Energy Accumulator) Algorithm**:

---

### **How the Dynamic Solution Works**

```
 [Vibration Pulses]  --->  [ + Energy Pool ]  ---> (Energy >= 30.0) ---> 🚨 EARTHQUAKE ALERT
                                  |
                                  v
                        [ Continuous Decay ]
                        ( -12.0 points/sec )
```

1. **Accidental Bump / Single Knock**:
   * Generates only 2–3 quick pulses $\implies$ adds $\sim 4-6$ energy points.
   * The natural decay immediately drains the energy back to zero in milliseconds.
   * **Result:** **No false alarm.**

2. **Real Seismic Shaking / Prolonged Earthquake**:
   * Continuous shaking generates a rapid torrent of pulses ($> 15-20$ pulses in under 1 second).
   * Energy generation easily exceeds the decay rate and crosses the **`SEISMIC_TRIGGER_LIMIT = 30.0`**.
   * **Result:** **Instant, verified Earthquake Alert!**

---

### **Configurable Parameters in Code**

If you ever want to fine-tune the stiffness or sensitivity:
* **`SEISMIC_TRIGGER_LIMIT = 30.0`**: Higher = Requires even harder/longer shaking; Lower = More sensitive.
* **`ENERGY_DECAY_RATE = 12.0`**: How fast accidental bump energy is forgotten per second.

---

### 👤 Alwin T R (`2026-09-10T12:18:46Z`)

chasnge the vibration value to default

---

### ⚡ Automated Actions (`2026-09-10T12:18:46Z`)

- **write_to_file**: Revert vibration to default

---

### ⚡ Automated Actions (`2026-09-10T12:18:55Z`)

- **run_command**: Compile and upload to ESP32

---

### 🤖 AI Engineering Assistant (`2026-09-10T12:19:03Z`)

I am compiling and uploading the sketch with the default vibration values to your ESP32.

---

### 🤖 AI Engineering Assistant (`2026-09-10T12:20:05Z`)

### **Reverted to Default Settings & Uploaded to ESP32!**

The vibration detection parameters have been reset back to the standard default values:

---

### **Default Settings in Effect:**
* **Earthquake Threshold**: `5 pulses` within `500 ms`
* **Earthquake Hold Duration**: `3 seconds`
* **Flood Detection Threshold**: `0.5 L/min` (with `3-second` latch)
* **Screen Refresh**: `200 ms`

---

### **LCD Display Summary:**

* **Normal / Idle**:
  ```text
  +------------------+
  |  STATUS: SAFE    |
  | MONITORING...    |
  +------------------+
  ```

* **Earthquake Sensed**:
  ```text
  +------------------+
  |** EARTHQUAKE **  |
  |   DETECTED!      |
  +------------------+
  ```

* **Flood / Flow Sensed**:
  ```text
  +------------------+
  |** FLOOD ALERT ** |
  |   DETECTED!      |
  +------------------+
  ```

---

### 👤 Alwin T R (`2026-09-10T12:21:15Z`)

no detect vibration

---

### ⚡ Automated Actions (`2026-09-10T12:21:15Z`)

- **write_to_file**: Update sketch for instant vibration detection

---

### ⚡ Automated Actions (`2026-09-10T12:21:23Z`)

- **run_command**: Compile and upload to ESP32

---

### 🤖 AI Engineering Assistant (`2026-09-10T12:21:31Z`)

I am compiling and uploading the ultra-sensitive vibration detection sketch to your ESP32.

---

### 🤖 AI Engineering Assistant (`2026-09-10T12:22:26Z`)

### **Ultra-Sensitive Vibration Detection Uploaded to ESP32!**

The program is now set to **instant 1-pulse detection on `CHANGE` interrupt mode** so that even the slightest touch, tap, or tilt on **GPIO 26** will immediately trip the Earthquake alert.

---

### **If your sensor is still not responding, check the hardware module:**

#### 1. **Check the Onboard LED on the Sensor Module**
* Most vibration sensor modules (SW-420) have two small LEDs:
  * **Power LED (`PWR` / `VCC`)**: Always ON.
  * **Signal / Detection LED (`D0` / `L2`)**: Should **flash or turn ON/OFF** when you tap or shake the sensor.
* If the `D0` LED **never flashes** when tapping the sensor, the module's sensitivity potentiometer is misadjusted.

#### 2. **Adjust the Blue Trimpot on the Sensor Module**
* Take a small flathead screwdriver and **turn the small brass screw on the blue box clockwise or counter-clockwise**:
  * Turn until the `D0` indicator LED is right on the edge of turning on when you tap the table.
  * Once the `D0` LED blinks when tapped, the ESP32 will instantly display:
    ```text
    +------------------+
    |** EARTHQUAKE **  |
    |   DETECTED!      |
    +------------------+
    ```

#### 3. **Verify Wire Connections**
* **VCC** $\rightarrow$ **`3.3V`** (or `5V` depending on module)
* **GND** $\rightarrow$ **`GND`**
* **DO (Digital Out)** $\rightarrow$ **`GPIO 26`** *(Make sure it is connected to `DO`, not `AO`)*

---

### 👤 Alwin T R (`2026-09-10T12:38:41Z`)

give code

---

### 🤖 AI Engineering Assistant (`2026-09-10T12:38:41Z`)

Here is the complete, full Arduino C++ code currently uploaded and running on your ESP32:

```cpp
#include <Arduino.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// ---------------- Pin Definitions ----------------
const int FLOW_SENSOR_PIN      = 27;  // Water flow sensor (Yellow wire)
const int VIBRATION_SENSOR_PIN = 26;  // Vibration / Tilt sensor (DO pin)
const int BUZZER_PIN           = 25;  // Active Buzzer (+)
const int ALERT_LED_PIN        = 2;   // Onboard status LED

// ---------------- I2C LCD Settings ----------------
// Default address is 0x27 on GPIO 21 (SDA) & GPIO 22 (SCL)
LiquidCrystal_I2C lcd(0x27, 16, 2);

// ---------------- Threshold Settings ----------------
const float CALIBRATION_FACTOR   = 7.5;   // YF-S201 sensor factor
const float FLOOD_FLOW_THRESHOLD = 0.5;   // Liters/min threshold for Flood Alert
const int   VIBRATION_THRESHOLD  = 1;     // Ultra-responsive: 1 pulse triggers alert
const unsigned long ALERT_HOLD_MS = 3000; // Hold alert on screen for 3 seconds

// ---------------- Volatile ISR Variables ----------------
volatile unsigned long flowPulseCount = 0;
volatile unsigned int vibrationPulseCount = 0;
volatile unsigned long lastVibrationTime = 0;

// ---------------- Global State Variables ----------------
bool earthquakeActive = false;
bool floodActive      = false;
unsigned long earthquakeAlertStartTime = 0;
unsigned long floodAlertStartTime      = 0;

float flowRate = 0.0;
unsigned long previousMillis = 0;
const unsigned long INTERVAL = 100; // Fast 100ms screen refresh

// ---------------- Interrupt Service Routines ----------------
void IRAM_ATTR flowPulseISR() {
  flowPulseCount++;
}

// Triggers on any pulse transition (HIGH-to-LOW or LOW-to-HIGH)
void IRAM_ATTR vibrationISR() {
  vibrationPulseCount++;
  lastVibrationTime = millis();
}

void setup() {
  Serial.begin(115200);

  // Initialize I2C Bus on GPIO 21 (SDA) and GPIO 22 (SCL)
  Wire.begin(21, 22);

  // Initialize LCD
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("DISASTER MONITOR");
  lcd.setCursor(0, 1);
  lcd.print("SYSTEM READY...");

  // Configure Pins
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(ALERT_LED_PIN, OUTPUT);
  pinMode(FLOW_SENSOR_PIN, INPUT_PULLUP);
  pinMode(VIBRATION_SENSOR_PIN, INPUT_PULLUP);

  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(ALERT_LED_PIN, LOW);

  // Attach Hardware Interrupts
  attachInterrupt(digitalPinToInterrupt(FLOW_SENSOR_PIN), flowPulseISR, RISING);
  attachInterrupt(digitalPinToInterrupt(VIBRATION_SENSOR_PIN), vibrationISR, CHANGE);

  delay(1500);
  lcd.clear();
  previousMillis = millis();
}

// ---------------- Non-Blocking Alarm Controller ----------------
void handleAlarms(unsigned long currentMillis) {
  if (earthquakeActive && floodActive) {
    // Dual Danger: Continuous Siren & Rapid Flash
    digitalWrite(BUZZER_PIN, HIGH);
    digitalWrite(ALERT_LED_PIN, (currentMillis / 100) % 2);
  } 
  else if (earthquakeActive) {
    // Earthquake: Fast Staccato Beep (50ms ON / 50ms OFF)
    bool beepState = (currentMillis % 100) < 50;
    digitalWrite(BUZZER_PIN, beepState ? HIGH : LOW);
    digitalWrite(ALERT_LED_PIN, beepState ? HIGH : LOW);
  } 
  else if (floodActive) {
    // Flood: Slow Pulsing Tone (200ms ON / 200ms OFF)
    bool beepState = (currentMillis % 400) < 200;
    digitalWrite(BUZZER_PIN, beepState ? HIGH : LOW);
    digitalWrite(ALERT_LED_PIN, beepState ? HIGH : LOW);
  } 
  else {
    // Safe / Normal: Silence
    digitalWrite(BUZZER_PIN, LOW);
    digitalWrite(ALERT_LED_PIN, LOW);
  }
}

void loop() {
  unsigned long currentMillis = millis();

  // 1. Detect Earthquake (Instant trigger on >= 1 pulse)
  if (vibrationPulseCount >= VIBRATION_THRESHOLD) {
    earthquakeActive = true;
    earthquakeAlertStartTime = currentMillis;
    vibrationPulseCount = 0;
    Serial.println("🚨 [ALERT] Earthquake / Vibration Detected!");
  }

  // Clear Earthquake Alert after 3 seconds of calm
  if (earthquakeActive && (currentMillis - earthquakeAlertStartTime > ALERT_HOLD_MS)) {
    earthquakeActive = false;
  }

  // 2. Flow Rate Check & Flood Alert Trigger
  if (currentMillis - previousMillis >= INTERVAL) {
    unsigned long duration = currentMillis - previousMillis;
    previousMillis = currentMillis;

    // Read Flow Pulses safely
    noInterrupts();
    unsigned long currentFlowPulses = flowPulseCount;
    flowPulseCount = 0;
    interrupts();

    // Calculate Flow Rate
    flowRate = ((1000.0 / duration) * currentFlowPulses) / CALIBRATION_FACTOR;

    // Trigger Flood Alert on water flow
    if (flowRate >= FLOOD_FLOW_THRESHOLD || currentFlowPulses > 0) {
      floodActive = true;
      floodAlertStartTime = currentMillis;
      Serial.printf("🌊 [ALERT] Flood Detected! Flow Rate: %.2f L/min\n", flowRate);
    }

    // Clear Flood Alert after 3 seconds of no flow
    if (floodActive && (currentMillis - floodAlertStartTime > ALERT_HOLD_MS)) {
      floodActive = false;
    }

    // 3. Update LCD Screen based on priority
    if (earthquakeActive && floodActive) {
      lcd.setCursor(0, 0);
      lcd.print("! DUAL DANGER ! ");
      lcd.setCursor(0, 1);
      lcd.print("QUAKE + FLOODING");
    } 
    else if (earthquakeActive) {
      lcd.setCursor(0, 0);
      lcd.print("** EARTHQUAKE **");
      lcd.setCursor(0, 1);
      lcd.print("   DETECTED!    ");
    } 
    else if (floodActive) {
      lcd.setCursor(0, 0);
      lcd.print("** FLOOD ALERT **");
      lcd.setCursor(0, 1);
      lcd.print("   DETECTED!    ");
    } 
    else {
      lcd.setCursor(0, 0);
      lcd.print("  STATUS: SAFE  ");
      lcd.setCursor(0, 1);
      lcd.print(" MONITORING...  ");
    }
  }

  // 4. Play Alarm Sounds & LED feedback
  handleAlarms(currentMillis);
}
```

---

### **Pin Connections Quick Reference**

| Component | Pin | ESP32 GPIO |
| :--- | :--- | :--- |
| **I2C 16x2 LCD** | `SDA` / `SCL` | **GPIO 21** / **GPIO 22** |
| | `VCC` / `GND` | **`VIN` (5V)** / **`GND`** |
| **Water Flow Sensor** | Signal (Yellow) | **GPIO 27** |
| | `VCC` / `GND` | **`VIN` (5V)** / **`GND`** |
| **Vibration / Tilt Sensor** | `DO` (Digital Out) | **GPIO 26** |
| | `VCC` / `GND` | **`3.3V`** / **`GND`** |
| **Active Buzzer** | `(+)` / `(-)` | **GPIO 25** / **`GND`** |

---


---
*Archived in [developer-chat-history](https://github.com/ALWINTR/developer-chat-history)*
