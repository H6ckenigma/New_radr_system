<div align="center">

# 🚗 Average Speed Detection Radar

**A Python simulation of a section-control speed enforcement system.**

Two radar checkpoints, one known distance, and a little arithmetic — that's all it takes
to catch a speeding car without ever tracking it.

[![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?logo=python&logoColor=white)](https://www.python.org/)
[![PyQt6](https://img.shields.io/badge/GUI-PyQt6-41CD52?logo=qt&logoColor=white)](https://pypi.org/project/PyQt6/)
[![Pandas](https://img.shields.io/badge/Data-Pandas-150458?logo=pandas&logoColor=white)](https://pandas.pydata.org/)
[![Status](https://img.shields.io/badge/status-simulation-orange)]()
[![License](https://img.shields.io/badge/license-MIT-blue)](LICENSE)


<img width="910" height="736" alt="image" src="https://github.com/user-attachments/assets/bfcfe3a7-58b7-498b-9d44-14446e12c6e6" />


</div>

---

> [!IMPORTANT]
> This is a **software simulation**. It does not talk to real radar hardware, does not
> identify real vehicles, and does not issue real traffic violations. Recording a real
> highway isn't something you can just do, so the radar input is generated synthetically —
> which is exactly what makes the processing logic easy to test.

---

## 💡 The Idea

Instant-speed radars measure how fast you're going *at one point*. Section control
(average speed enforcement) does something smarter: it measures how fast you were going
**on average** over a whole stretch of road — so braking for the camera doesn't help.

All it needs is four pieces of information:

1. The distance between two checkpoints
2. A plate + timestamp at point **A**
3. A plate + timestamp at point **B**
4. Accurate clocks

<img width="1024" height="572" alt="image" src="https://github.com/user-attachments/assets/36c6ef6f-ae4e-4a6f-a0a7-98bb64e2fedf" />


| Distance | Travel Time | Average Speed | Speed Limit | Status |
| -------- | ----------- | ------------- | ----------- | ------ |
| 50 km | 30 min | 100 km/h | 120 km/h | 🟢 Good |
| 50 km | ~21 min | 140 km/h | 120 km/h | 🔴 High Speed |

> [!NOTE]
> The processor **never reads the simulated car's speed directly**. It reconstructs the
> speed from the two timestamps, exactly like a real system would. The generated speed
> exists only so the simulator knows when to place the Radar B detection.

---

## 🚀 Quick Start

```bash
# 1. Clone
git clone https://github.com/H6ckenigma/Avearge-speed-detection-radar-source-code.git
cd Avearge-speed-detection-radar-source-code

# 2. Virtual environment (Python 3.10+)
python -m venv .venv
source .venv/bin/activate        # Linux / macOS
.venv\Scripts\activate           # Windows

# 3. Dependencies
pip install -r requirements.txt
pip install -r src/requirements-gui.txt   # only if you want the GUI
```

### 🖥️ Run with the GUI

```bash
cd src
python gui.py
```

Press **Start Sim**. Detections stream into the log, speeding vehicles are highlighted,
and an alert sound fires on each violation.

### 💻 Run in the terminal

```bash
cd src
python main.py
```

```text
Car #12 > 583921|F|15 going 137 km/h >> Speeding!!
```

Results are written continuously to `data/results.csv`. Stop with <kbd>Ctrl</kbd> + <kbd>C</kbd>.

> **Windows shortcut:** double-click `Run_radar.bat`.

---

## 🧠 How It Works

The project is a small data pipeline in four stages.

### 1️⃣ Data Simulator — `src/Data_simulator.py`

Stands in for the physical world. For each vehicle it:

- generates a Moroccan-style license plate (`123456|F|15`)
- picks a speed
- stamps a Radar A detection time
- computes the theoretical travel time for that speed over the configured distance
- stamps the matching Radar B detection time
- appends both rows to the radar CSV files

```
🚗 Vehicle
   │
   ├─> Generate plate ──> Generate speed ──> Radar A timestamp
   │                                              │
   │                                    travel time = distance ÷ speed
   │                                              ▼
   └────────────────────────────────────> Radar B timestamp
                                                  │
                                                  ▼
                                       write radar_a.csv / radar_b.csv
```

### 2️⃣ The Radars — `data/radar_a.csv`, `data/radar_b.csv`

Two independent detection logs, each holding a plate and a timestamp. They deliberately
know nothing about each other — just like two real roadside units.

### 3️⃣ Processor — `src/Processor.py`

The engine of the project:

1. Read Radar A and Radar B logs
2. Match vehicles by license plate
3. Compute the travel time from the two timestamps
4. Compute the average speed (`distance ÷ time`)
5. Compute the excess over the limit
6. Resolve the city from the plate's regional code
7. Classify the vehicle (`Good` / `High Speed`)
8. Write everything to `data/results.csv`

### 4️⃣ Interface — `src/gui.py`

A PyQt6 desktop app that lets you:

| | |
| --- | --- |
| ▶️ Start / ⏹️ Stop the simulation | 🚨 See speeding alerts highlighted |
| 🚗 Watch detections live | 🔊 Play an alert sound |
| ⚙️ Inspect the active configuration | 🧹 Clear the log |
| 📏 Change the radar distance | 🚦 Change the speed limit |

---

## 🏗️ Architecture

<img width="1024" height="572" alt="image" src="https://github.com/user-attachments/assets/ec5858fe-2710-4cc0-be63-fd30f78bae85" />


## ⚙️ Configuration

Everything lives in `src/radar_conf.py`:

```python
Distance   = 50.0   # km between Radar A and Radar B
Speed_limit = 120   # km/h — above this a vehicle is flagged "High Speed"
Num_cars    = 30    # configured number of simulated vehicles
```

| Setting | Meaning |
| ------- | ------- |
| `Distance` | Length of the monitored section, in kilometres. |
| `Speed_limit` | Maximum allowed **average** speed over that section. |
| `Num_cars` | Configured vehicle count. The loop can run continuously, so treat this as a configuration/interface value rather than a hard cap. |

### 🔬 Things worth trying

Change the config, rerun, and diff the CSVs — it's the fastest way to see the pipeline
behave:

```python
Distance = 100.0   # long section, strict limit
Speed_limit = 90
```

```python
Distance = 20.0    # short section, permissive limit
Speed_limit = 130
```

Then compare `data/radar_a.csv`, `data/radar_b.csv` and `data/results.csv` side by side.

---

## 📊 Understanding the Output

`data/results.csv`:

```text
plate           speed   excess   city    status
123456|F|15     100     0        Fès     Good
654321|A|6      137     17       ...     High Speed
```

| Field | Description |
| ----- | ----------- |
| `plate` | Simulated license plate |
| `speed` | Average speed reconstructed from the two timestamps |
| `excess` | How far above the limit (`0` when compliant) |
| `city` | City resolved from the plate's regional code |
| `status` | `Good` or `High Speed` |


<img width="910" height="736" alt="image" src="https://github.com/user-attachments/assets/648ad5e2-5d49-46c6-81cb-ec943c850c3b" />


---

## 📁 Project Structure

```
.
├── data/
│   ├── radar_a.csv          # simulated Radar A detections
│   ├── radar_b.csv          # simulated Radar B detections
│   └── results.csv          # processed vehicle results
│
├── docs/
│   └── images/              # screenshots used in this README
│
├── src/
│   ├── Data_simulator.py    # generates vehicles + radar detections
│   ├── Processor.py         # matching, speed calculation, classification
│   ├── main.py              # continuous CLI simulation
│   ├── gui.py               # PyQt6 interface
│   ├── radar_conf.py        # configuration
│   ├── requirements-gui.txt # GUI dependencies
│   └── beep.wav             # alert sound
│
├── requirements.txt
├── runtime.txt
├── Run_radar.bat            # Windows launcher
└── README.md
```

---



## ⚠️ Limitations

This is an experimental project, not enforcement software. Known gaps:

- Simulated radar data instead of physical sensors
- CSV files instead of a database
- Minimal error handling; no automated tests
- No persistent vehicle/session management
- Plates are generated, not read by a camera
- Average speed only covers the section between the two checkpoints
- No production-grade security or reliability guarantees

Each of these is a decent starting point for the next version.

---

## 🗺️ Roadmap

- [ ] **🔌 Hardware interface** — swap the simulator for real radar/camera input behind a sensor abstraction
- [ ] **📷 ALPR/ANPR** — automatic plate recognition from camera frames
- [ ] **🗄️ Database** — SQLite or PostgreSQL instead of CSV
- [ ] **🌐 Web dashboard** — browser-based monitoring and traffic statistics
- [ ] **🚗 Robust matching** — handle missed detections, duplicate plates, repeat trips, malformed rows
- [ ] **📊 Analytics** — speed distribution, traffic volume, time-of-day patterns, detection history
- [ ] **🧪 Test suite** — speed math, timestamp parsing, matching, classification, invalid input

---

## 🛠️ Built With

| Tool | Role |
| ---- | ---- |
| 🐍 **Python** | Core language |
| 🐼 **Pandas** | CSV handling and data processing |
| 🖥️ **PyQt6** | Desktop interface |
| 🕐 **datetime** | Timestamps and travel-time arithmetic |

---

## 📜 Disclaimer

Educational and experimental use only. Deploying anything like this for real enforcement
would require certified hardware, legal compliance, calibration, validation, security
review and reliability testing — none of which this project has.

---

## 📄 License

Released under the MIT License. See [`LICENSE`](LICENSE) for details.

---

<div align="center">

**Built by [H6ckenigma](https://github.com/H6ckenigma)** · Have fun experimenting 🚗💨

</div>
