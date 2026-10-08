<div align="center">

# AI-Based Smart Vehicle Lane Detection and Control System

**Real-time lane detection with OpenCV and automatic vehicle steering using ESP32 microcontrollers.**

![Python](https://img.shields.io/badge/Python-3.8%2B-3776AB?logo=python&logoColor=white)
![Flask](https://img.shields.io/badge/Flask-Backend-000000?logo=flask&logoColor=white)
![OpenCV](https://img.shields.io/badge/OpenCV-Computer%20Vision-5C3EE8?logo=opencv&logoColor=white)
![ESP32](https://img.shields.io/badge/ESP32-Firmware-E7352C?logo=espressif&logoColor=white)

</div>

---

## Table of Contents

- [Overview](#overview)
- [System Architecture](#system-architecture)
- [Project Structure](#project-structure)
- [Hardware Requirements](#hardware-requirements)
- [Wiring Guide](#wiring-guide)
- [Quick Start (No Hardware)](#quick-start-no-hardware)
- [Hardware Setup](#hardware-setup)
- [Lane Detection Pipeline](#lane-detection-pipeline)
- [Vehicle Control Logic](#vehicle-control-logic)
- [Dashboard](#dashboard)
- [Log Format](#log-format)
- [Troubleshooting](#troubleshooting)
- [Dependencies](#dependencies)

---

## Overview

This project uses a camera mounted on a vehicle to detect road lanes and automatically steer the vehicle. A Flask backend processes the video feed with OpenCV, determines which lane the vehicle occupies, and sends direction commands over Wi-Fi to an ESP32-WROOM that drives the motors.

It is designed to be beginner-friendly: the entire pipeline can be tested on a laptop webcam before any hardware is connected.

---

## System Architecture

```
┌──────────────┐        ┌───────────────────────┐        ┌──────────────────┐
│  ESP32-CAM   │  Wi-Fi │  Computer             │  HTTP  │  ESP32-WROOM     │
│              │ ─────► │  Flask + OpenCV       │ ─────► │                  │
│ Video stream │        │  Lane detection       │        │  L298N driver    │
│ (on vehicle) │        │  Local preview window │        │  DC motors       │
└──────────────┘        └───────────────────────┘        └──────────────────┘
```

**Data flow**

1. The **ESP32-CAM** captures live road video and streams it over Wi-Fi.
2. **Flask + OpenCV** read the stream, detect lanes, and show a local preview window.
3. The **Flask backend** decides a direction: `LEFT`, `RIGHT`, or `FORWARD`.
4. An **HTTP command** is sent to the **ESP32-WROOM**, which drives the motors.

---

## Project Structure

```
car_truck_2/
├── app.py                  # Flask backend (main server)
├── lane_detection.py       # OpenCV lane detection logic
├── esp32_controller.py     # Sends commands to ESP32-WROOM
├── requirements.txt        # Python dependencies
│
├── templates/
│   ├── index.html          # Vehicle selection homepage
│   └── dashboard.html      # Control dashboard
│
├── static/
│   ├── style.css           # Dark theme styling
│   └── script.js           # Dashboard JavaScript
│
└── esp32/
    ├── ESP32_WROOM/
    │   └── ESP32_WROOM.ino # Motor controller firmware
    └── ESP32_CAM/
        └── ESP32_CAM.ino   # Camera stream firmware
```

---

## Hardware Requirements

| Component            | Purpose                      |
| -------------------- | ---------------------------- |
| ESP32-CAM            | Camera mounted on the vehicle |
| ESP32-WROOM          | Motor controller              |
| L298N motor driver   | Drives the DC motors          |
| 2× DC motors         | Drive the vehicle wheels      |
| Power bank / battery | Powers the vehicle            |
| USB cable            | Programs the ESP32 boards     |

---

## Wiring Guide

### L298N → ESP32-WROOM

| L298N Pin | ESP32 GPIO | Function          |
| --------- | ---------- | ----------------- |
| IN1       | GPIO 14    | Motor A direction |
| IN2       | GPIO 27    | Motor A direction |
| ENA       | GPIO 12    | Motor A speed     |
| IN3       | GPIO 26    | Motor B direction |
| IN4       | GPIO 25    | Motor B direction |
| ENB       | GPIO 13    | Motor B speed     |
| GND       | GND        | Common ground     |
| 5V        | 5V         | Logic power       |

### DC Motors → L298N

| Motor       | L298N Terminals |
| ----------- | --------------- |
| Left wheel  | OUT1, OUT2      |
| Right wheel | OUT3, OUT4      |

---

## Quick Start (No Hardware)

Test the full flow with your laptop webcam first. No ESP32 boards are needed, and all commands are printed to the terminal instead of moving motors.

**1. Install dependencies**

```bash
cd car_truck_2
pip install -r requirements.txt
```

**2. Enable test mode**

In `esp32_controller.py`:

```python
TEST_MODE = True            # No real hardware needed
```

In `lane_detection.py`:

```python
USE_WEBCAM_FOR_TEST = True  # Uses your laptop camera
```

**3. Run the server**

```bash
python app.py
```

**4. Open the dashboard**

Go to **http://127.0.0.1:5000**, then:

1. Select **CAR** or **TRUCK** on the homepage.
2. Click **START** on the dashboard. A local OpenCV window opens with live lane detection.
3. Watch the log panel update in real time.
4. Click **STOP** to end the session.

---

## Hardware Setup

### A. Flash the ESP32-CAM

1. Open `esp32/ESP32_CAM/ESP32_CAM.ino` in the Arduino IDE.
2. Set your Wi-Fi credentials:
   ```cpp
   const char* WIFI_SSID     = "YOUR_WIFI_NAME";
   const char* WIFI_PASSWORD = "YOUR_PASSWORD";
   ```
3. Select the board **AI-Thinker ESP32-CAM** and upload the sketch.
4. Open the Serial Monitor at **115200 baud**, press Reset, and note the IP address shown.
5. Update `lane_detection.py`:
   ```python
   ESP32_CAM_URL = "http://YOUR_CAM_IP/stream"
   USE_WEBCAM_FOR_TEST = False
   ```

### B. Flash the ESP32-WROOM

1. Open `esp32/ESP32_WROOM/ESP32_WROOM.ino` in the Arduino IDE.
2. Set your Wi-Fi credentials:
   ```cpp
   const char* WIFI_SSID     = "YOUR_WIFI_NAME";
   const char* WIFI_PASSWORD = "YOUR_PASSWORD";
   ```
3. Select the board **ESP32 Dev Module** and upload the sketch.
4. Open the Serial Monitor and note the IP address shown.
5. Update `esp32_controller.py`:
   ```python
   ESP32_WROOM_IP = "YOUR_WROOM_IP"
   TEST_MODE = False
   ```

---

## Lane Detection Pipeline

The road is divided into three lanes:

```
|    LEFT    |   MIDDLE   |   RIGHT    |
0           w/3          2w/3          w
```

| Step | Operation        | Description                                          |
| ---- | ---------------- | ---------------------------------------------------- |
| 1    | Crop             | Keep the bottom 55% of the frame (road area only)    |
| 2    | HSV conversion   | Convert to HSV colour space                          |
| 3    | Black mask       | Detect dark lane markings with `cv2.inRange`         |
| 4    | Gaussian blur    | Reduce noise                                         |
| 5    | Canny edges      | Find line boundaries                                 |
| 6    | Trapezoid mask   | Ignore sky and walls                                 |
| 7    | Hough transform  | Find lane lines                                      |
| 8    | Line averaging   | Produce one left and one right boundary              |
| 9    | Midpoint         | Calculate the vehicle's position                     |
| 10   | Classification   | Label the position as `LEFT`, `MIDDLE`, or `RIGHT`   |

---

## Vehicle Control Logic

### Car (prefers the right lane)

| Current Lane | Command Sent |
| ------------ | ------------ |
| LEFT         | RIGHT        |
| MIDDLE       | RIGHT        |
| RIGHT        | FORWARD      |

### Truck (prefers the left lane)

| Current Lane | Command Sent |
| ------------ | ------------ |
| RIGHT        | LEFT         |
| MIDDLE       | LEFT         |
| LEFT         | FORWARD      |

---

## Dashboard

| Feature          | Description                                  |
| ---------------- | -------------------------------------------- |
| Vehicle Status   | Shows `RUNNING` or `STOPPED`                 |
| Current Lane     | Shows `LEFT`, `MIDDLE`, or `RIGHT`           |
| Direction        | Shows the current command (← → ↑ ■)          |
| Signal Logs      | Live log of all events and commands          |
| START button     | Starts detection and opens the OpenCV window |
| STOP button      | Stops detection and the motors               |
| OVERTAKE button  | Sends an `OVERTAKE` command to the ESP32     |

> **Note:** The camera feed is not shown in the browser. It opens as a **local OpenCV window** on the computer running the server.

---

## Log Format

```
[INFO]    CAR STARTED
[INFO]    CAMERA CONNECTED
[INFO]    CAR is in LEFT lane
[COMMAND] RIGHT
[INFO]    CAR is in RIGHT lane – MAINTAIN
[COMMAND] FORWARD
[INFO]    CAR STOPPED
```

---

## Troubleshooting

| Problem                       | Solution                                                                                   |
| ----------------------------- | ------------------------------------------------------------------------------------------ |
| `Camera could not be opened`  | Check the ESP32-CAM IP, or set `USE_WEBCAM_FOR_TEST = True`.                               |
| `Cannot connect to ESP32`     | Check the ESP32-WROOM IP, or set `TEST_MODE = True`.                                       |
| Dashboard not loading         | Make sure Flask is running with `python app.py`.                                           |
| No lanes detected             | Improve lighting and adjust `BLACK_LOWER` / `BLACK_UPPER` in `lane_detection.py`.          |
| OpenCV window not showing     | Make sure you are not running in a headless environment.                                   |

---

## Dependencies

| Package         | Purpose                           |
| --------------- | --------------------------------- |
| `flask`         | Web server and routing            |
| `opencv-python` | Camera capture and lane detection |
| `numpy`         | Math and array operations         |
| `requests`      | HTTP calls to the ESP32           |

Install everything with:

```bash
pip install -r requirements.txt
```
