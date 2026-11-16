# DRDO Sensor Communication System

A modular sensor-monitoring prototype developed during a summer internship in the SDD division at DRDO (DRDE), Gwalior. The project integrates sensors with different communication protocols into one monitoring workflow:

**TCP/UDP sensors → Python Universal Converter → MQTT → Python monitoring backend → WebSocket → Electron/React dashboard**

It was built as a learning-to-implementation project covering serial/RS-485 concepts, TCP, UDP, MQTT, Python sockets, `asyncio`, React, Electron, Raspberry Pi edge deployment concepts, and Windows packaging experiments.

> This is an educational/research project repository. It is not an official DRDO product or release.

## What the system does

- Reads data from configured sensor sources.
  - Bio and Chemical sources use TCP request/response communication.
  - FCAD uses UDP and a binary-packet decoder.
- Uses one configuration-driven Python converter for multiple sensor instances.
- Publishes readings through an MQTT broker using location-based topics.
- Subscribes to all sensor topics, validates/parses payloads, and saves daily raw-data logs.
- Sends normalized live data to a local WebSocket server.
- Displays locations, sensor status, readings, live SVG trends, and an Add Sensor form in an Electron/React dashboard.

## Architecture

Bio/Chem TCP sensors or simulators ─┐
                                    ├─> Universal_Convertor.py ─> MQTT broker
FCAD UDP sensor ────────────────────┘                                 │
                                                                       ▼
                                                              Subscriber.py
                                                     parse + alarm + raw daily logs
                                                                       │
                                                          WebSocket (localhost)
                                                                       ▼
                                                        Electron + React dashboard

The dashboard deliberately does not connect to MQTT or edit `Config.txt` itself. `Subscriber.py` is the boundary between MQTT and the UI: it owns parsing, raw logging, alarm decisions, WebSocket broadcasts, and validated configuration updates requested by the UI.

## Current implementation

The current implementation is in:

Final Project v1 - Code/
├── Main Project v4/          # Python backend, converter, config, simulators
└── Sensor Dashboard/         # React/Vite/Electron desktop dashboard

Earlier folders under `Backend Versions/` and `Prerequisites/` are retained to show the development journey. They are useful learning material, but they are not the current runtime by default.

## Repository structure

DRDO-Sensor-Communication-System/
├── Final Project v1 - Code/
│   ├── Main Project v4/
│   │   ├── Config.txt
│   │   ├── Convertors/
│   │   │   ├── Universal_Convertor.py
│   │   │   └── Start_All_Converters.bat
│   │   ├── Monitoring Station/
│   │   │   └── Subscriber.py
│   │   ├── Virtual Sensors/
│   │   │   ├── Bio_Main_Gate_01.py
│   │   │   ├── Bio_Director_Office_01.py
│   │   │   ├── Chem_Main_Gate_01.py
│   │   │   ├── Chem_Director_Office_01.py
│   │   │   └── Start_All_Virtual_Sensors.bat
│   │   └── requirements.txt
│   └── Sensor Dashboard/
│       ├── electron/
│       ├── src/
│       ├── package.json
│       └── vite.config.js
├── Backend Versions/         # Historical backend snapshots
├── Prerequisites/            # Protocol, UI, graph, Pi, and packaging learning work
├── Other Reports/            # Project reports and progress documentation
├── Report_Abstract.docx
└── Report_Detailed.docx

## Technology stack

| Layer | Technology | Responsibility |
| --- | --- | --- |
| Sensor communication | TCP, UDP, Python sockets | Acquires device/simulator data |
| Conversion | Python, Paho MQTT | Converts/publishes sensor readings |
| Messaging | Mosquitto, MQTT | Routes topic-based sensor data |
| Monitoring backend | Python, `asyncio`, `websockets` | Parses, logs, and bridges MQTT to the UI |
| Dashboard | React, Vite, SVG | Renders live readings, status, and graphs |
| Desktop shell | Electron | Runs the dashboard as a desktop app |

## MQTT topic convention

Every sensor publishes to a predictable location-based topic:

sensors/<location>/<type>/<sensor-id>

Examples:

sensors/main-gate/bio/bio-main-gate-01
sensors/director-office/chem/chem-director-office-01

`Subscriber.py` subscribes to `sensors/#`, which receives all sensor topics in this hierarchy.

## Configuration

`Final Project v1 - Code/Main Project v4/Config.txt` is JSON configuration with three sections:

{
  "global": {
    "ip_broker": "127.0.0.1",
    "port_broker": 1883,
    "broker_keep_alive": 60
  },
  "sensor_types": {
    "bio": { "protocol": "tcp", "parser": "passthrough" },
    "chem": { "protocol": "tcp", "parser": "passthrough" },
    "fcad": { "protocol": "udp", "parser": "fcad_bytes" }
  },
  "sensors": {
    "bio-main-gate-01": {
      "type": "bio",
      "location": "main-gate",
      "ip": "127.0.0.1",
      "port": 5050,
      "request": "send bio-main-gate-01",
      "request_interval": 3
    }
  }
}
```

`sensor_types` defines reusable protocol behavior. `sensors` defines individual physical or simulated instances. Add a sensor instance to configuration instead of duplicating converter code.

> The included Bio/Chem endpoints use `127.0.0.1` and are intended for the bundled virtual-sensor development setup. Replace them with real network addresses only in the configuration used for your deployment.

## Supported sensor paths

### Bio and Chemical

The converter:

1. opens a TCP connection,
2. sends the configured request followed by a newline,
3. reads exactly one newline-terminated response,
4. publishes the response to MQTT, and
5. reconnects after connection or decoding failures.

Bio data is expected as six CSV fields. Chemical data is expected as five CSV fields.

### FCAD

The converter sends a UDP discovery/request packet, validates the response source IP, and decodes documented binary fields into JSON before publishing. Known packet fields include unit identity, selected G/H values, pressure, temperatures, battery, and status fields.

Some later packet bytes remain intentionally undecoded and are kept as `unknownTail`. Their meaning should not be assumed without approved OEM protocol documentation and hardware validation.

## Run locally - development setup

### Prerequisites

- Python 3.10+ recommended
- Node.js 18+ recommended
- Mosquitto MQTT broker
- Windows is the easiest environment for the included `.bat` launchers

### 1. Start the MQTT broker

Install and start Mosquitto. The supplied local configuration expects it at:

127.0.0.1:1883

For a remote broker or Raspberry Pi-based converter, update `global.ip_broker` in the deployment configuration and ensure the broker is reachable from that device.

### 2. Install Python dependencies

cd "Final Project v1 - Code\Main Project v4"
python -m pip install -r requirements.txt

### 3. Start the virtual Bio/Chem sensors

From `Final Project v1 - Code\Main Project v4\Virtual Sensors`:

.\Start_All_Virtual_Sensors.bat

This opens four TCP simulator windows:

- Bio - Main Gate (`5050`)
- Chem - Main Gate (`5051`)
- Bio - Director Office (`5052`)
- Chem - Director Office (`5053`)

### 4. Start the converters

From `Final Project v1 - Code\Main Project v4\Convertors`:

.\Start_All_Converters.bat

This starts the four configured Bio/Chem converter processes. For FCAD, start the appropriate converter separately after configuring a reachable FCAD device:

python Universal_Convertor.py fcad-main-gate-01

### 5. Start the monitoring backend

From `Final Project v1 - Code\Main Project v4`:

python ".\Monitoring Station\Subscriber.py"

The backend subscribes to MQTT and starts a local WebSocket server at:

ws://127.0.0.1:8765

Daily raw payload logs are written to:

log/<location>/<type>/<sensor-id>_<YYYY-MM-DD>.txt

### 6. Start the Electron dashboard

In a new terminal:

cd "Final Project v1 - Code\Sensor Dashboard"
npm install
npm run dev

Vite and Electron start together. Wait for the Electron desktop window; the dashboard is designed to talk to the already-running local Python backend.

## Dashboard features

- Building/location overview
- List of all configured sensors
- Sensor detail tabs with endpoint, topic, readings, and last-received time
- Live custom SVG graphs, retaining up to 60 readings per sensor in browser memory
- Status LEDs:
  - **Green** - a reading arrived within the previous 10 seconds
  - **Red** - an online Bio sensor reports alarm `Y`
  - **Grey** - no recent reading
- Add Sensor form with frontend feedback and authoritative backend validation
- Automatic WebSocket reconnect every two seconds after an unexpected disconnect

The current backend generates alarms only for Bio readings with `alarm == "Y"`. Chemical and FCAD readings are displayed with no alarm rule until approved thresholds are defined.

## Add a sensor

The dashboard can add a sensor instance to `Config.txt` through the backend. The backend validates the ID, predefined type, location, IPv4-like address format, port, request text, and polling interval before writing the file.

Adding a sensor does **not** automatically deploy or start a converter. After adding an instance, start the converter manually or provision it through an approved deployment process.

## Project evolution

The repository intentionally keeps the learning and iteration history:

1. Serial/RS-485 foundations and TCP socket practice
2. MQTT publisher/subscriber model with virtual Bio/Chem sensors
3. UDP and binary FCAD packet integration
4. Universal converter and JSON configuration
5. Location-based sensor instances and MQTT topics
6. Subscriber-to-WebSocket bridge and React/Electron monitoring UI
7. Raspberry Pi deployment and Windows packaging experiments

## Current limitations

- The repository's verified development workflow requires Mosquitto, `Subscriber.py`, and the dashboard to be started separately.
- The final dashboard source does not include a verified `electron-builder` configuration or code that starts a packaged Python backend. Treat installer notes and historical package artefacts as separate work requiring rebuild and testing from the final source.
- Raspberry Pi/systemd material is documented in prerequisite notes; a production-ready service unit and deployment manifest should be versioned and tested before operational use.
- MQTT and local WebSocket authentication/TLS are not implemented in the current development flow.
- Sensor configuration writes do not yet include locking, backups/versioning, audited changes, or endpoint-collision checks.
- The dashboard history is in memory only; it is not a historical database.
- Some FCAD packet bytes need OEM confirmation before they can be interpreted safely.
- A single broker and Subscriber instance are assumed; high availability is not implemented.

## Suggested next steps

1. Add automated parser, FCAD-decoder, configuration, and WebSocket contract tests.
2. Add deliberate MQTT reconnection/error handling and atomic, backed-up configuration writes.
3. Version actual Raspberry Pi service files and deployment instructions.
4. Build and test a reproducible standalone Windows package from the final source.
5. Move to a managed MQTT server only with authentication, TLS where required, firewall rules, monitoring, and rollback planning.
6. Add durable storage and historical visualization/reporting.
7. Define FCAD and Chemical alarm thresholds with domain/OEM approval.

## Documentation

- `Report_Abstract.docx` - project overview
- `Report_Detailed.docx` - technical project report
- `Other Reports/` - milestone and subsystem reports
- `Prerequisites/` - learning exercises and prototype work that led to the final architecture

## Author

**Tejas**  
B.Tech Computer Science Engineering, Third Year  
Samrat Ashok Technological Institute, Vidisha

---

If you use or extend this repository, begin with the final-code folders above, verify the relevant report claim against the implementation, and test each layer of the data path before connecting real hardware.
