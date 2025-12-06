# Raspberry-Pico-W-Read-Ultrasonik-Sensor-Send-to-Thingspeak
This setup for Raspberry Pico W handling data to Thingspeak.

# MicPySer Dual-Mode: IoT & Offline Ultrasonic Monitoring

Proyek ini menggunakan **Raspberry Pi Pico W** untuk membaca **sensor ultrasonik HC-SR04** dan dapat beroperasi dalam **dua mode**:  

1. **IoT Mode**: Mengirim data jarak ke Thingspeak jika WiFi tersambung.  
2. **Offline Mode**: Jika WiFi mati atau internet down, Pico tetap membaca sensor secara lokal dengan indikator LED.

LED built-in (GPIO25) digunakan untuk menandai mode:  
- **Kedip lambat** → IoT Mode aktif  
- **Kedip cepat** → Offline Mode aktif  

---

## Hardware

- Raspberry Pi Pico W  
- HC-SR04 Ultrasonic Sensor  
- LED Built-in Pico W (GPIO25)  

### Wiring HC-SR04
| HC-SR04 | Pico W  |
|----------|---------|
| VCC      | 3.3V    |
| GND      | GND     |
| TRIG     | GP15    |
| ECHO     | GP14    |

---

## Kode Micropython

```python
import network
import time
from machine import Pin, time_pulse_us
import urequests

# ----- Konfigurasi ----- #
SSID = "nama_wifi"
PASSWORD = "password_wifi"
THINGSPEAK_URL = "https://api.thingspeak.com/update?api_key=YOUR_API_KEY&field1={}"
LED = Pin(25, Pin.OUT)
TRIG = Pin(15, Pin.OUT)
ECHO = Pin(14, Pin.IN)
INTERVAL = 5  # detik

# ----- Fungsi Ultrasonik ----- #
def read_distance():
    TRIG.value(0)
    time.sleep_us(2)
    TRIG.value(1)
    time.sleep_us(10)
    TRIG.value(0)
    duration = time_pulse_us(ECHO, 1, 30000)  # timeout 30ms
    distance = (duration / 2) / 29.1  # cm
    return round(distance, 1)

# ----- Fungsi LED ----- #
def led_blink(mode):
    if mode == "iot":
        LED.value(1)
        time.sleep(0.2)
        LED.value(0)
        time.sleep(1)
    else:  # offline
        LED.value(1)
        time.sleep(0.2)
        LED.value(0)
        time.sleep(0.2)

# ----- Cek WiFi ----- #
def connect_wifi():
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    if not wlan.isconnected():
        print("Connecting to WiFi...")
        wlan.connect(SSID, PASSWORD)
        timeout = 10
        while not wlan.isconnected() and timeout > 0:
            time.sleep(1)
            timeout -= 1
    return wlan.isconnected()

# ----- Loop Utama ----- #
while True:
    distance = read_distance()
    if connect_wifi():
        # Mode IoT
        led_blink("iot")
        try:
            response = urequests.get(THINGSPEAK_URL.format(distance))
            response.close()
            print("IoT mode: Distance sent =", distance, "cm")
        except:
            print("IoT error, fallback offline")
    else:
        # Mode Offline / Embedded
        led_blink("offline")
        print("Offline mode: Distance =", distance, "cm")
