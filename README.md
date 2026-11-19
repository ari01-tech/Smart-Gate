# SmartGate – Smart Parking Gate System

A prototype smart gate system that combines an ESP32-CAM, a Python/Flask server, license-plate text recognition, face detection, and an Arduino serial connection.

## Project overview

The current source code is organized around these components:

- **ESP32-CAM (`CAM.ino`)** – connects to Wi-Fi, captures a camera frame when it receives the `TAKE_PHOTO` serial command, and sends the JPEG image to the Python server.
- **Python backend (`server.py`)** – receives images, detects faces, reads text that may represent a license plate using EasyOCR, checks the plate against an allowlist, records each decision in a CSV file, and emits an event to the web dashboard using Flask-SocketIO.
- **Dashboard styling (`style.css`)** – provides a dark, responsive card-based interface.

> **Important:** This README describes the behavior present in the supplied source files. The dashboard template (`templates/dashboard.html`) was referenced by the backend but was not included with the supplied files.

## Features in the supplied code

- ESP32-CAM image capture and HTTP upload.
- Face detection using `face_recognition`.
- Text recognition using EasyOCR.
- Access decision based on both the recognized plate and face detection:
  - `allowed` when the plate is in the configured allowlist and a face is detected.
  - `denied` otherwise.
- CSV event logging in `logs.csv`.
- Live image/event updates through Flask-SocketIO.
- Serial communication with an Arduino-compatible board.
- Manual gate command event through the Socket.IO event `control_gate`.

## Repository structure

```text
.
├── CAM.ino
├── server.py
├── style.css
├── templates/
│   └── dashboard.html   # Required by server.py; not included in the supplied files
├── captured_faces/      # Created by server.py at runtime
└── logs.csv             # Created by server.py at runtime
```

## Hardware

The supplied firmware indicates that the project uses:

- ESP32-CAM (the code comments refer to the AI Thinker pin layout).
- An Arduino-compatible board connected to the Python host over serial.
- A computer running the Python server.
- A Wi-Fi network shared by the ESP32-CAM and the server.

The exact gate actuator, wiring, power requirements, and complete camera pin configuration are not documented in the supplied files. Confirm these against your actual hardware before powering the system.

## Software requirements

Python packages imported by `server.py`:

```bash
pip install flask flask-cors flask-socketio opencv-python easyocr face-recognition numpy requests pyserial
```

You may also need the appropriate system build tools and dependencies for `face_recognition` and its `dlib` dependency. Installation steps vary by operating system.

For the ESP32-CAM:

- Arduino IDE or another ESP32-compatible build environment.
- ESP32 board support package.
- The `esp_camera`, `WiFi`, and `HTTPClient` libraries provided by the ESP32 Arduino core.

## Configuration

Update the configuration values in the source files before running the project. Do not commit real Wi-Fi passwords or private network details to a public repository.

### 1. ESP32-CAM (`CAM.ino`)

Set your Wi-Fi credentials and the server address:

```cpp
const char* ssid = "YOUR_WIFI_SSID";
const char* password = "YOUR_WIFI_PASSWORD";
String serverURL = "http://YOUR_SERVER_IP:5000/capture";
```

The server IP must be reachable from the ESP32-CAM over the same network.

### 2. Python server (`server.py`)

Review and update:

```python
camera_ip = "http://YOUR_ESP32_CAM_IP:80/stream"
arduino_port = "COM6"  # Change to the serial port used by your system
allowed_plates = ["YOUR_ALLOWED_PLATE_1", "YOUR_ALLOWED_PLATE_2"]
```

- `camera_ip` is used by the `/video_feed` endpoint to read the ESP32-CAM stream.
- `arduino_port` must match the port assigned by your operating system (for example, `COM6` on Windows or a `/dev/tty...` path on Linux).
- Replace the sample plate values with the plates you want to authorize.

**Network note:** The supplied ESP32-CAM and Python files contain different private IP subnets/addresses. Configure both to use the correct addresses on your actual network.

## Running the Python server

1. Connect the Arduino-compatible board and identify its serial port.
2. Configure the ESP32-CAM Wi-Fi credentials and server URL.
3. Configure the ESP32-CAM stream address, serial port, and allowed plates in `server.py`.
4. Install the Python dependencies.
5. Ensure the required `templates/dashboard.html` file exists.
6. Start the backend:

```bash
python server.py
```

The Flask-SocketIO server listens on port **5000** and binds to `0.0.0.0`.

Open the dashboard from a browser on the same network using:

```text
http://YOUR_SERVER_IP:5000/
```

The `/` route renders `templates/dashboard.html`, so the dashboard will not load until that template is provided.

## Backend routes and events

| Route / event | Method / type | Purpose |
|---|---|---|
| `/` | HTTP GET | Renders `templates/dashboard.html`. |
| `/video_feed` | HTTP GET | Streams frames from the configured ESP32-CAM stream URL. |
| `/capture` | HTTP POST | Accepts a JPEG image, performs face detection and OCR, decides access, logs the result, and emits a `new_car` event. |
| `control_gate` | Socket.IO event | Accepts a command in `data["command"]` and forwards it to the serial device when connected. |
| `new_car` | Socket.IO event | Broadcasts the recognized plate, face status, access decision, and processed image (Base64). |
| `status` | Socket.IO event | Broadcasts a status message after a manual gate command. |

### `/capture` response

The endpoint returns one of the following text responses:

- `allowed`
- `denied`
- `error`

The access rule in the supplied backend is:

```python
access = "allowed" if plate in allowed_plates and face_status == "found" else "denied"
```

## Generated files

When the backend runs, it creates:

- `captured_faces/` – created at startup. The supplied code currently does not save detected face images into this folder.
- `logs.csv` – access log with the columns `Time`, `Plate`, `Face`, and `Access`.

## Current implementation notes

Please review these items before treating the prototype as a complete working system:

1. **ESP32-CAM camera configuration:** `CAM.ino` declares `camera_config_t config` but the supplied file does not assign its required pin, clock, pixel-format, frame-size, or other configuration fields before calling `esp_camera_init(&config)`. Complete the configuration for the exact ESP32-CAM board.
2. **Serial protocol mismatch to resolve:** `CAM.ino` waits for `TAKE_PHOTO` and then posts a captured image to `/capture`. The supplied `server.py`, however, writes `allowed` or `denied` to the serial port after processing an image, and its manual-control handler writes the supplied command. These message flows do not currently form a complete, clearly defined two-way gate-control protocol. Align the firmware and backend commands before testing the gate.
3. **ESP32-CAM stream:** The backend expects an HTTP MJPEG stream at `camera_ip`. Confirm that the ESP32-CAM firmware or another camera service actually exposes `/stream` on the configured address; the supplied `CAM.ino` only shows the capture-and-upload flow.
4. **Dashboard template:** `dashboard.html` is referenced but was not among the supplied files. The HTML and Socket.IO client code are needed to display events and send manual-control commands.
5. **OCR accuracy:** The current code selects the first OCR result whose cleaned text length is between 5 and 10 characters. It does not validate a country-specific license-plate format.
6. **Security:** The development configuration uses a hard-coded Flask secret, unrestricted CORS, and unauthenticated HTTP/Socket.IO control paths. Add authentication, restrict allowed origins, use secure secrets, and protect the gate-control interface before deployment.
7. **Privacy and safety:** Face and vehicle data can be sensitive. Define appropriate consent, retention, access-control, and deletion practices. Test the gate mechanism safely and provide a manual override.

## Troubleshooting

- **Cannot connect to ESP32-CAM:** Check Wi-Fi credentials, IP address, subnet, and network reachability.
- **Arduino serial port cannot open:** Check the selected port, USB cable, drivers, and whether another application is using the port.
- **Dashboard page fails:** Confirm `templates/dashboard.html` exists in the expected location.
- **Camera initialization fails:** Complete the board-specific `camera_config_t` configuration and verify the selected board.
- **No OCR or face result:** Check image quality, lighting, camera focus, and the installed EasyOCR / `face_recognition` dependencies.

## License

No license was specified in the supplied source files. Add a `LICENSE` file before publishing or distributing this project.
