# Telemetry GUI — Real-Time Formula Student Dashboard

A real-time telemetry dashboard for **Team Fateh's Formula Student car**, written in **Processing (Java)**. It reads an **18-channel CSV data stream over serial at 230400 baud**, draws it on a **1920×1080** track-side display, overlays a live **FPV camera feed**, and logs every received frame to **CSV** for post-session analysis.

The repo also includes an **Arduino signal simulator** (`TelemetryV1.0.ino`) that emits frames in the same format, so the GUI can be developed and tested on the bench without the car.

---

## Demo

<video src="assets/demo.mp4" controls muted width="100%"></video>

[Watch the demo video](assets/demo.mp4)

---

## Features

| Area | What it does |
|---|---|
| **Serial ingestion** | Reads newline-terminated CSV frames at **230400 baud** and parses **18 channels** per frame. A frame is only accepted if it has **exactly 18 fields**, so malformed or partial packets never reach the display. |
| **Live gauges** | Arc gauges for **RPM (0–11,000)** and **Speed (0–150)**; big-number readouts for **Gear** (0 shows `N`), **Engine Temp (°C)**, **Battery Voltage (V)** and **Steering Angle (°)**. |
| **Bar indicators** | **Brake pressure** and **Throttle** (0–100 %), plus **4 per-corner channels** (FL / FR / RL / RR, 0–150). |
| **Status lights** | Green/red indicators for **Radiator (R)** and **Data Logging (D)**. |
| **Rolling time-series plots** | **7 traces** in sliding-window buffers: Speed + RPM, Throttle + Brake pressure + Brake temp (**385 samples each**), and lateral/longitudinal acceleration (**240 samples each**, ±3 G). |
| **G-G (friction-circle) plot** | Plots live X/Y acceleration against **1 G / 2 G / 3 G** reference rings to show grip usage and weight transfer. |
| **Raw data panel** | All **18 channels** shown as numbers for quick debugging. |
| **FPV camera feed** | Live video from an onboard/USB camera (Processing `video` library), shown alongside the telemetry. |
| **CSV logging** | Every raw frame is appended to a CSV file, which is flushed and closed when the sketch exits. |

---

## Data frame format

Each frame is one line of **18 comma-separated values**, terminated by `\n`:

| Index | Channel | Type / range | Shown as |
|---:|---|---|---|
| 0 | On time | ms since boot (`millis()`) | Raw data |
| 1 | Engine RPM | 0 – 11,000 | Gauge + rolling plot |
| 2 | Engine temperature | °C | Readout |
| 3 | Gear | int (0 = Neutral) | Readout (`N`) |
| 4 | Speed | 0 – 150 | Gauge + rolling plot |
| 5 | Brake pressure | 0 – 100 | Bar + rolling plot |
| 6 | Battery voltage | V | Readout |
| 7 | Radiator | 0 / 1 | Status light **R** |
| 8 | Data logging | 0 / 1 | Status light **D** |
| 9 | Throttle | 0 – 100 % | Bar + rolling plot |
| 10 | Brake temperature | 0 – 120 °C | Rolling plot |
| 11 | Front Left | 0 – 150 | Bar |
| 12 | Front Right | 0 – 150 | Bar |
| 13 | Rear Left | 0 – 150 | Bar |
| 14 | Rear Right | 0 – 150 | Bar |
| 15 | Accelerometer X | −3 G … +3 G | G-G plot + rolling plot |
| 16 | Accelerometer Y | −3 G … +3 G | G-G plot + rolling plot |
| 17 | Steering angle | degrees | Readout |

Example frame:

```
10423,6500,80,4,88.5,35,12.4,1,1,62,95,70,71,69,70,0.8,-1.2,30
```

---

## How it works

```
 Car sensors ─► Microcontroller ─► Serial / wireless link ─► Processing GUI ─┬─► 1080p dashboard
 (or the Arduino simulator)         (230400 baud, 18-ch CSV)                  └─► CSV log file
                                                                  USB camera ─► FPV feed
```

1. `draw()` runs once per frame. If serial data is waiting, it reads one line up to `\n`.
2. The raw line is **written to the CSV log** first, then split on `,`.
3. If there are exactly **18 fields**, they are converted to floats and replace the current values.
4. Each rolling buffer shifts left by one and the newest value is appended at the end, which gives a scrolling plot.
5. Gauges, bars, plots, the G-G plot, the raw data panel and the camera frame are redrawn.

---

## Repository structure

| File | Purpose |
|---|---|
| `TelemetryV1_0.pde` | The Processing dashboard: serial parsing, drawing, video and CSV logging. |
| `TelemetryV1.0.ino` | Arduino simulator: reads 2 potentiometers (A0, A1) and emits 18-channel test frames. |
| `sketch.properties` | Processing sketch metadata. |

---

## Getting started

### Requirements
- [Processing 4](https://processing.org/download) (3.x also works)
- The **Video Library for Processing** (install it from *Sketch → Import Library → Manage Libraries*)
- A serial data source: the car's telemetry receiver, or an Arduino running `TelemetryV1.0.ino`
- (Optional) a USB/FPV camera
- A 1920×1080 display is recommended, because the layout uses fixed pixel positions

### Configure
Edit these lines in `TelemetryV1_0.pde` to match your machine:

```java
video  = new Capture(this, Capture.list()[3]);                // camera index
myPort = new Serial(this, Serial.list()[0], 230400);          // serial port index + baud
output = createWriter("C:\\...\\Data1.csv");                  // CSV log path
```

Tip: run `printArray(Serial.list());` and `printArray(Capture.list());` once to find the correct indices.

### Run
1. Connect the data source (and camera).
2. Open `TelemetryV1_0.pde` in Processing and press **Run**.
3. Close the sketch window normally so the CSV log is flushed and saved.

### Bench testing with the Arduino simulator
1. Wire two potentiometers to **A0** and **A1**.
2. In `TelemetryV1.0.ino`, set the baud rate to match the GUI: `Serial.begin(230400);` (the file currently uses `9600`).
3. Upload the sketch and run the GUI. Turning the pots will drive RPM, speed, throttle, brake and the per-corner channels. The other channels are fixed test values.

---

## Known limitations
- The layout uses absolute pixel coordinates designed for **1920×1080**.
- The serial port, camera index and CSV path are hard-coded.
- The simulator sends the raw ADC value (0–1023) for acceleration, while the GUI expects values in **±3 G**. Scale the value on the sender side for realistic G-G plots.
- Only one frame is read per `draw()` call. With very high sender rates, the serial buffer can fall behind the display.

---

## Context
Built for **Team Fateh (Formula Student, TIET Patiala)** as the track-side visualisation layer of the car's telemetry system. It helped engineers check vehicle health in real time and review driver inputs (throttle, braking, steering, G-forces) alongside onboard video after each run.
