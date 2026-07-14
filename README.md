# Autonomous Plant-Watering Robot

A robot that finds a plant pot on its own, drives up to it, checks if the soil is dry, and stops when it gets there. It runs on three microcontrollers talking to each other over WiFi. An ESP32-CAM spots the pot, a main ESP32 handles navigation and hosts a web dashboard, and a small ESP-01S just reads soil moisture.

<img width="1920" height="1080" alt="ESP32 - CAM" src="https://github.com/user-attachments/assets/6080a9ba-1da1-40f2-b860-97cb1a73b71d" />

---

## How it works

The three boards talk to each other over WiFi on the same local network. The main ESP32 is basically the brain. It runs the web server, pulls in data from the other two boards, and decides how the robot moves. Everything connects through mDNS, so you can always reach the robot at `http://wateringrobot.local` without needing to look up its IP.

---

## Modules

### 1. Main ESP32: Robot controller

This is the central unit. Navigation runs on core 1, while a separate FreeRTOS task on core 0 keeps polling the camera module for the target angle every 150 ms.

The robot works as a simple state machine with four states:

| State | Behaviour |
|---|---|
| `SEARCH` | Spins left or right in short bursts, pausing between turns to see if the camera has spotted anything |
| `FOLLOWING` | Drives toward the pot using proportional steering. The further off-centre the target is, the harder the inner wheel gets pushed |
| `WALL_AVOID` | Backs up for 500 ms, then turns right for 550 ms if it hits something that isn't the pot |
| `ARRIVED` | Stops and nudges itself into alignment with short 150 ms spin pulses until the pot is centred in frame |

Obstacle detection runs off an HC-SR04 ultrasonic sensor at the front. If something shows up within 18 cm, the robot checks whether the camera has seen the pot in the last 5 seconds. If it has, that "something" is the pot, so the robot switches to `ARRIVED`. If not, it treats it as a wall and backs off.

There's also a live dashboard on the root URL that refreshes every second, showing the current state, distance, camera angle, soil moisture, and whether the pot was recently seen.

---

### 2. ESP32-CAM: Visual target detection

An AI Thinker ESP32-CAM running a simple colour detection algorithm. It grabs frames at QQVGA (160x120) in RGB565 and checks every 4th pixel for red.

A pixel counts as red if its red channel is above 12 and higher than both green and blue. Once more than 6 red pixels show up in a frame, it works out the horizontal centre of mass of those pixels and turns that into an angle based on the camera's field of view (65° by default):

```
angle = (centerX - 80) × (FOV/2) / 80
```

That angle (negative for left, positive for right, or 404 if nothing's found) gets sent to the main ESP32 over HTTP every 200 ms. The main board uses it both to steer while following and to confirm arrival when the ultrasonic sensor picks something up.

https://github.com/user-attachments/assets/d0a08f9c-38ac-41e1-830b-2962d2f2c1c2

---

### 3. ESP-01S: Soil moisture sensor

A stripped-down ESP8266 board that checks a digital soil moisture sensor on GPIO2 every 10 seconds and reports back to the main ESP32:

```
http://wateringrobot.local/soil?status=1   // dry
http://wateringrobot.local/soil?status=0   // wet
```

HIGH means dry, LOW means wet. Shows up on the dashboard and can be used to decide whether the plant actually needs watering once the robot arrives.

---

## System diagram

```
┌─────────────┐     HTTP /angle     ┌──────────────────┐
│  ESP32-CAM  │ ─────────────────▶  │                  │
│  Red object │                     │   Main ESP32     │
│  detection  │                     │   Robot + Web    │
└─────────────┘                     │   Server         │
                                    │                  │
┌─────────────┐     HTTP /soil      │                  │
│   ESP-01S   │ ─────────────────▶  │                  │
│   Soil      │                     └──────────────────┘
│   moisture  │
└─────────────┘
```

---

## Hardware

| Component | Role |
|---|---|
| ESP32 (main) | Navigation, state machine, web server |
| ESP32-CAM (AI Thinker) | Red object detection, angle calculation |
| ESP-01S | Soil moisture sensing |
| HC-SR04 | Obstacle and arrival detection |
| L298N | Dual H-bridge motor driver |
| 4x DC gear motors | Drive |
| Capacitive soil moisture sensor | Plant soil reading |

---

## Tech stack

- Arduino framework (ESP32 + ESP8266 core)
- FreeRTOS for dual-core task pinning on the ESP32
- mDNS for zero-config device discovery on the local network
- HTTP for communication between boards
- RGB565 frame processing, computer vision done on-device with no external libraries
