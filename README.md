# unoQ_bridge_tutorial
A tutorial that gives a quick over view into the working of unoQ bridge router using a simple ultrasonic sensor.


## Architecture Overview

The system operates across two main processing domains:

1. **Microcontroller Unit (MCU - C++):** Handles real-time, time-critical hardware operations (ultrasonic pulse triggering, high-precision timing via `pulseIn()`).
2. **Microcomputer Unit (Linux MPU - Python):** Runs high-level logic, polls data across the internal RPC bus via `RouterBridge`, exposes REST endpoints, and manages WebSockets to serve a web interface.

```
+-----------------------------------------------------------------------+
|                             ARDUINO BOARD                             |
|                                                                       |
|  +--------------------+   RPC Bridge    +--------------------------+  |
|  |     STM32 MCU      | <=============> |        Linux MPU         |  |
|  |     (sketch.ino)   |  (RouterBridge) |        (main.py)         |  |
|  +---------+----------+                 +------------+-------------+  |
|            |                                         |                |
+------------|-----------------------------------------|----------------+
             |                                         |
             v                                         v
     [ HC-SR04 Sensor ]                       [ Web Browser Client ]
   Trig: Pin 9 / Echo: Pin 10                 WebSocket / HTTP (5000)

```

---

## Level 1: Standalone Microcontroller Code

This initial stage runs purely on the microcontroller. It uses standard GPIO bit-banging to send ultrasonic triggers and measure reflections.

### Code (`sketch.ino`)

```cpp
const int trigPin = 9;
const int echoPin = 10;

long duration;
int distance;

void setup() {
  Serial.begin(9600);
  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);
}

void loop() {
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);
  
  duration = pulseIn(echoPin, HIGH);
  distance = duration * 0.034 / 2;
  
  Serial.print("Distance: ");
  Serial.print(distance);
  Serial.println(" cm");
  
  delay(500);
}

```

### Technical Explanation

* **`digitalWrite(trigPin, HIGH)` for 10µs:** Sends an ultrasonic burst from the HC-SR04 module at 40 kHz.
* **`pulseIn(echoPin, HIGH)`:** Measures the time in microseconds for the echo pin to remain HIGH (i.e., time taken for sound to travel to the target and back).
* **`distance = duration * 0.034 / 2`:** Calculates distance using the speed of sound in air ($\approx 0.034\text{ cm/µs}$). The value is divided by 2 to account for the round-trip distance.

---

## Level 2: Python Integration via RouterBridge

This level introduces cross-domain communication. The C++ sketch exposes a function via `Bridge.provide()`, enabling the Linux side (Python) to call it remotely via RPC (Remote Procedure Call).

### Microcontroller Code (`sketch.ino`)

```cpp
#include <Arduino_RouterBridge.h>

const int trigPin = 9;
const int echoPin = 10;

long duration;
int distance;

// Function exposed to Python via RouterBridge
int get_distance() {
  return distance;
}

void setup() {
  Serial.begin(9600);
  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);

  // Initialize bridge communication
  Bridge.begin();
  Bridge.provide("get_distance", get_distance);
}

void loop() {
  // Keep bridge communications active
  Bridge.update();

  // Distance measurement logic
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);
  
  duration = pulseIn(echoPin, HIGH);
  distance = duration * 0.034 / 2;
  
  Serial.print("Distance: ");
  Serial.print(distance);
  Serial.println(" cm");
  
  delay(500);
}

```

### Python Script (`main.py`)

```python
import time
from arduino.app_utils import App, Bridge

def main():
    print("Ultrasonic Sensor Monitor Started")
    
    while True:
        try:
            # Poll C++ for the current distance value
            dist_cm = int(Bridge.call("get_distance"))
            print(f"Distance: {dist_cm} cm")
        except Exception as e:
            print(f"Bridge Error: {e}")
        
        # Poll every 500ms to match loop delay
        time.sleep(0.5)

if __name__ == '__main__':
    main()

```

### Technical Explanation

* **`#include <Arduino_RouterBridge.h>`:** Loads internal inter-processor RPC IPC definitions.
* **`Bridge.begin()` & `Bridge.update()`:** Starts the IPC link and processes pending requests on every MCU loop iteration.
* **`Bridge.provide("get_distance", get_distance)`:** Registers `get_distance()` in the RPC lookup table so external processors can query it.
* **`Bridge.call("get_distance")`:** Sends a request from Python to the MCU over internal serial, executing `get_distance()` on the MCU and returning the `int` value back to Python.

---

## Level 3: Full Web UI System

This production phase adds a Web application layer using `WebUI` (`socket.io` and `REST`). Python polls the sensor in a background thread and streams data live to the browser.

### Project Structure

```text
├── sketch.ino
└── web/
    ├── main.py
    ├── index.html
    ├── app.js
    └── style.css

```

---

### Code & Detailed Breakdown

#### 1. Microcontroller Code (`sketch.ino`)

```cpp
#include <Arduino_RouterBridge.h>

const int trigPin = 9;
const int echoPin = 10;

long duration;
int distance;

int get_distance() {
  return distance;
}

void setup() {
  Serial.begin(9600);
  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);

  Bridge.begin();
  Bridge.provide("get_distance", get_distance);
}

void loop() {
  Bridge.update();

  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);
  
  duration = pulseIn(echoPin, HIGH);
  distance = duration * 0.034 / 2;
  
  Serial.print("Distance: ");
  Serial.print(distance);
  Serial.println(" cm");
  
  delay(500);
}

```

#### 2. Linux App Script (`web/main.py`)

```python
import time
import threading
from arduino.app_utils import App, Bridge
from arduino.app_bricks.web_ui import WebUI

ui = WebUI()
current_distance = 0

def poll_sensor():
    """Background thread polling MCU for distance and broadcasting updates."""
    global current_distance
    while True:
        try:
            val = int(Bridge.call("get_distance"))
            current_distance = val
            ui.send_message("distance_update", {"distance": val})
        except Exception as e:
            print(f"Bridge Error: {e}")
        time.sleep(0.5)

# REST Endpoint
ui.expose_api("GET", "/api/distance", lambda: {"distance": current_distance})

# WebSocket Event Handlers
def on_get_initial_state(client, data):
    ui.send_message("distance_update", {"distance": current_distance}, client)

ui.on_message("get_initial_state", on_get_initial_state)

# Start background sensor polling thread
poll_thread = threading.Thread(target=poll_sensor, daemon=True)
poll_thread.start()

# Start application (blocks until stopped)
App.run()

```

* **Thread Isolation (`threading.Thread`):** Polling the bridge happens in a background thread so it doesn't block the `App.run()` event loop.
* **WebSocket Broadcast (`ui.send_message`):** Sends JSON updates containing `{ "distance": val }` to all connected clients over Socket.IO under the topic `distance_update`.
* **REST API (`ui.expose_api`):** Exposes a `GET /api/distance` endpoint that returns current metrics as a JSON response.

#### 3. Web View Structure (`web/index.html`)

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Distance Monitor</title>
    <link rel="stylesheet" type="text/css" href="style.css" />
  </head>
  <body>
    <div class="card">
      <h2>Ultrasonic Sensor</h2>
      <div class="distance-display">
        <span id="dist-val">--</span>
        <span class="unit">cm</span>
      </div>
      <p class="status">Live Distance Reading</p>
      <div id="error-container" class="error-message" style="display: none"></div>
    </div>

    <script src="libs/socket.io.min.js"></script>
    <script src="libs/arduino.js"></script>
    <script src="app.js"></script>
  </body>
</html>

```

#### 4. Frontend Controller (`web/app.js`)

```javascript
const distVal = document.querySelector('#dist-val');
const errorContainer = document.querySelector('#error-container');

// Instantiate Arduino WebUI library wrapper
const ui = new WebUI();

ui.on_connect(() => {
  errorContainer.style.display = 'none';
  ui.send_message('get_initial_state');
});

ui.on_disconnect(() => {
  errorContainer.style.display = 'block';
  errorContainer.textContent = 'Connection to board lost.';
});

ui.on_message('distance_update', (data) => {
  if (data && data.distance !== undefined) {
    distVal.textContent = data.distance;
  }
});

```

* **Connection Lifecycle Handlers:** Automatically clears or renders diagnostic connection error elements when WebSocket status changes.
* **`get_initial_state` Payload Request:** Fetches the current distance reading immediately upon loading without waiting for the next update cycle.

#### 5. Stylesheet (`web/style.css`)

```css
body {
  font-family: Arial, sans-serif;
  background-color: #ecf1f1;
  color: #2c353a;
  padding: 40px;
  text-align: center;
}

.card {
  max-width: 350px;
  margin: 0 auto;
  background: white;
  padding: 30px;
  border-radius: 12px;
  box-shadow: 0 4px 10px rgba(0,0,0,0.08);
}

.distance-display {
  margin: 20px 0;
  color: #008184;
}

#dist-val {
  font-size: 64px;
  font-weight: bold;
}

.unit {
  font-size: 24px;
  color: #666;
  margin-left: 4px;
}

.status {
  font-size: 14px;
  color: #777;
}

.error-message {
  margin-top: 15px;
  padding: 10px;
  background: #f8d7da;
  color: #721c24;
  border-radius: 6px;
}

```

---

## How to Deploy and Run

1. Open **Arduino App Lab**.
2. Create a project named `UltrasonicMonitor`.
3. Paste `sketch.ino` into the MCU tab.
4. Place `main.py`, `index.html`, `app.js`, and `style.css` in the app directory.
5. Click **Run**.
6. Access the dashboard from any browser at `
