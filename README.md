# meterologystation
Here is guide how to create meterologystation with Blynk
# DIY IoT Meteorology Station (ESP32-S3 + Blynk)

A low-cost, WiFi-connected weather station built on the ESP32-S3 that measures temperature, humidity, barometric pressure, ambient light, and distance (for snow depth / water level), with live data pushed to the Blynk IoT dashboard.

## Features

- 🌡️ Temperature & humidity (DHT22 / AM2302)
- 🌬️ Barometric pressure (BMP180)
- ☀️ Ambient light level (LDR)
- 📏 Distance sensing for snow depth or water level (HC-SR04 ultrasonic)
- ☁️ Live dashboard via Blynk IoT (mobile app + web)
- 📊 Historical data charts (built into Blynk)
- 🔔 Configurable threshold alerts (optional, via Blynk Automations)

## Hardware Required

| Component | Purpose | Notes |
|---|---|---|
| ESP32-S3 Dev Board | Main controller | Confirm exact model — pin availability varies |
| DHT22 (AM2302) | Temperature & humidity | 10kΩ pull-up resistor if not built into your module |
| BMP180 | Barometric pressure | I2C, fixed address 0x77 |
| LDR (photoresistor) | Ambient light | Needs a fixed resistor for a voltage divider |
| HC-SR04 | Ultrasonic distance (snow depth / water level) | 5V logic — confirm your board's Echo pin is 5V tolerant or use a voltage divider |
| Fixed resistor (~10kΩ) | LDR voltage divider | Any resistor in the 1kΩ–10kΩ range works |
| Breadboard + jumper wires | Prototyping | — |
| USB-C cable | Programming & power | Data-capable, not charge-only |

## Wiring

| Sensor | Pin | ESP32-S3 GPIO |
|---|---|---|
| BMP180 | SDA | GPIO 8 |
| BMP180 | SCL | GPIO 9 |
| DHT22 | Data | GPIO 5 |
| LDR | Analog Out | GPIO 4 |
| HC-SR04 | Trig | GPIO 6 |
| HC-SR04 | Echo | GPIO 7 |

All sensors share 3.3V/5V and GND rails as appropriate to each module's spec sheet. **Always confirm your specific board's pinout diagram** — GPIO numbering can vary between ESP32-S3 board vendors.

## Software Setup

### 1. Install Arduino IDE
Download from [arduino.cc](https://www.arduino.cc/en/software).

### 2. Add ESP32 board support
File → Preferences → Additional Board Manager URLs, add:
```
https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
```
Then Tools → Board → Boards Manager → search "esp32" → install.

### 3. Select the correct board
Tools → Board → select the profile matching your exact chip (e.g. "ESP32S3 Dev Module" for S3-based boards). Using the wrong profile can cause unreliable uploads or incorrect default pin behavior.

### 4. Install required libraries
Via Library Manager (Sketch → Include Library → Manage Libraries):
- `DHT sensor library` (Adafruit)
- `Adafruit Unified Sensor`
- `SFE_BMP180`
- `Blynk` (latest, for Blynk IoT — not legacy Blynk Legacy)

### 5. Set up Blynk
1. Create a free account at [blynk.cloud](https://blynk.cloud)
2. Create a new Template, add datastreams for each sensor (V0–V5), matching units and ranges to what your code sends
3. Create a Device from the template, copy its **Auth Token**
4. Build a dashboard with Gauge widgets bound to each datastream

### 6. Configure the sketch
Open the `.ino` file and fill in:
```cpp
#define BLYNK_TEMPLATE_ID   "your_template_id"
#define BLYNK_TEMPLATE_NAME "your_template_name"
#define BLYNK_AUTH_TOKEN    "your_auth_token"

char ssid[] = "your_wifi_ssid";
char pass[] = "your_wifi_password";
```

### 7. Upload and verify
Upload the sketch, open Serial Monitor at **115200 baud**, and confirm sensor readings print correctly before checking the Blynk dashboard.

## Verifying Sensor Wiring

Run an I2C scanner sketch first to confirm BMP180 (0x77) is detected on the bus before troubleshooting further — this catches most wiring mistakes immediately.

## Troubleshooting

| Symptom | Likely Cause |
|---|---|
| Only some sensors show data | Check I2C bus init order — `Wire.begin()` must run before any sensor's `.begin()` call |
| LCD/BMP180 blank or unresponsive | Wrong I2C address, or contrast potentiometer needs adjusting (LCD only) |
| WiFi won't connect | Captive-portal network (common on campus guest WiFi) — test with a personal hotspot to confirm |
| Blynk shows old/delayed data | Check WiFi stability; confirm `Blynk.run()` and `timer.run()` are called every loop with no blocking delays |
| Gauge always shows max value | Datastream Min/Max range doesn't match the value range your code sends |
| LDR reads backwards (dark = 100%) | LDR is wired on the opposite side of the voltage divider — either re-wire or invert the `map()` call in code |

## Future Improvements

- Add a rain gauge (tipping-bucket + reed switch) for real rainfall measurement
- Add an anemometer/wind vane for wind speed and direction
- Add a UV index sensor
- SD card logging for offline data history
- Battery + solar power for fully off-grid operation
- Outdoor radiation shield enclosure to prevent direct sunlight from skewing temperature/humidity readings

```````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````````
// ============================================================
// BLYNK DEFINITIONS (Must stay at the very top of the sketch)
// ============================================================
#define BLYNK_TEMPLATE_ID   "TMPL6Qtgb7jUw"
#define BLYNK_TEMPLATE_NAME "Weather Monitoring System"
#define BLYNK_AUTH_TOKEN    "vCyObxczzPTMfn7GYfmHuaq-ecwLMYEM"
#define BLYNK_PRINT Serial

#include <WiFi.h>
#include <WiFiClient.h>
#include <BlynkSimpleEsp32.h>
#include <Wire.h>
#include <DHT.h>
#include <SFE_BMP180.h>

// ============================================================
// NETWORK CREDENTIALS
// ============================================================
char auth[] = BLYNK_AUTH_TOKEN;
char ssid[] = "";
char pass[] = "";

// ============================================================
// HARDWARE PIN DEFINITIONS (ESP32-S3)
// ============================================================
#define SDA_PIN 8      // BMP180 / LCD SDA
#define SCL_PIN 9      // BMP180 / LCD SCL
#define DHT_PIN 5      // AM2302 (DHT22) Data Pin
#define DHTTYPE DHT22
#define LDR_PIN 4      // LDR Analog Pin
#define TRIG_PIN 6     // HC-SR04 Trig
#define ECHO_PIN 7     // HC-SR04 Echo
#define MOUNT_HEIGHT_CM 100  // distance from sensor to ground with no snow — set to your actual mount height

// Sensor & Timer Objects
DHT dht(DHT_PIN, DHTTYPE);
SFE_BMP180 bmp;
BlynkTimer timer;

// ============================================================
// ULTRASONIC DISTANCE READ
// ============================================================
float readDistanceCM() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH, 30000); // 30ms timeout, ~5m max
  if (duration == 0) return -1; // no echo

  return duration * 0.0343 / 2; // speed of sound / 2 (round trip)
}

// ============================================================
// SENSOR READING & BLYNK TRANSMISSION FUNCTION
// ============================================================
void sendWeatherData() {
  Serial.print("--- SENSOR DATA READINGS ---\n");

  // --- 1. DHT22 ---
  float tempC = dht.readTemperature();
  float humidity = dht.readHumidity();

  if (!isnan(tempC)) {
    Blynk.virtualWrite(V0, tempC);
    Serial.printf("Temp (V0):     %.1f C\n", tempC);
  } else {
    Serial.println("AM2302 Read Error!");
  }

  if (!isnan(humidity)) {
    Blynk.virtualWrite(V1, humidity);
    Serial.printf("Humidity (V1): %.1f %%\n", humidity);
  }

  // --- 2. BMP180 ---
  char status;
  double bmpTemp, bmpPressure;
  bool bmpSuccess = false;

  status = bmp.startTemperature();
  if (status != 0) {
    delay(status);
    status = bmp.getTemperature(bmpTemp);
    if (status != 0) {
      status = bmp.startPressure(3);
      if (status != 0) {
        delay(status);
        status = bmp.getPressure(bmpPressure, bmpTemp);
        if (status != 0) bmpSuccess = true;
      }
    }
  }

  if (bmpSuccess) {
    Blynk.virtualWrite(V2, bmpPressure);
    Serial.printf("Pressure (V2): %.1f hPa\n", bmpPressure);
  } else {
    Serial.println("BMP180 Read Error!");
  }

  // --- 3. LDR (inverted: higher = brighter) ---
  int ldrRaw = analogRead(LDR_PIN);
  int lightPercent = map(ldrRaw, 0, 4095, 100, 0);
  Blynk.virtualWrite(V3, lightPercent);
  Serial.printf("Light (V3):    %d %%\n", lightPercent);

  // --- 4. Ultrasonic (snow depth) ---
  float distanceCM = readDistanceCM();
  if (distanceCM > 0) {
    float snowDepthCM = MOUNT_HEIGHT_CM - distanceCM;
    if (snowDepthCM < 0) snowDepthCM = 0;
    Blynk.virtualWrite(V5, snowDepthCM);
    Serial.printf("Snow Depth (V5): %.1f cm\n", snowDepthCM);
  } else {
    Serial.println("Ultrasonic Read Error!");
  }

  Serial.println();
}

// ============================================================
// SETUP
// ============================================================
void setup() {
  Serial.begin(115200);
  delay(1000);

  Wire.begin(SDA_PIN, SCL_PIN);

  dht.begin();

  if (bmp.begin()) {
    Serial.println("BMP180 initialized successfully.");
  } else {
    Serial.println("BMP180 initialization failed! Check SDA (GPIO 8) and SCL (GPIO 9).");
  }

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  analogReadResolution(12);

  WiFi.begin(ssid, "");
  Serial.print("Connecting to WiFi");
  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED && attempts < 20) {
    delay(500);
    Serial.print(".");
    attempts++;
  }

  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("\nWiFi connected!");
    Serial.print("IP address: ");
    Serial.println(WiFi.localIP());
  } else {
    Serial.println("\nWiFi FAILED to connect.");
  }

  Blynk.begin(auth, ssid, "", "blynk.cloud", 80);

  timer.setInterval(2000L, sendWeatherData);
}

// ============================================================
// MAIN LOOP
// ============================================================
void loop() {
  Blynk.run();
  timer.run();
}
