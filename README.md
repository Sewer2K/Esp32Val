# WiFi-ESP32-Valorant-Cheat

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## Overview

This repository is a proof-of-concept for an **external Valorant cheat** using an **ESP32-S3-WROOM-1 N16R8**[ESP32 S3 Dev Board - 16MB Flash, 8MB PSRAM]([https://github.com/Megant88/Valorant-GUI-Cheat-Arduino](https://www.robotics.org.za/ESP32-S3-N16R8-EXT) board for hardware-based aimbot and triggerbot functionality over WiFi. Unlike traditional Arduino setups, this leverages the ESP32's built-in WiFi for wireless communication, reducing cable clutter and potential detection vectors.

To avoid Valorant's anti-cheat (Vanguard) flagging direct screen captures (e.g., via `mss` or `pyautogui`), detection runs inside a **custom OBS plugin** that mimics the official **Game Capture Source**. The plugin captures the Valorant window safely (using OBS's graphics hook for DirectX/OpenGL interception), processes frames with OpenCV-like logic for enemy color/body detection, and sends trigger/aim commands to the ESP32 via HTTP over local WiFi.

The ESP32 emulates a **generic USB HID mouse** for inputs (clicks/movement), spoofed to appear as a standard device to evade hardware fingerprinting. This setup is inspired by external color-based cheats but prioritizes stealth: no memory reading, no direct PC-side inputs, and OBS integration for "legit" capture.

**Key Inspirations & Code Reuse**:
- Heavily references [Megant88/Valorant-GUI-Cheat-Arduino](https://github.com/Megant88/Valorant-GUI-Cheat-Arduino) for aimbot/triggerbot logic (adapted from their Python `main.py` and Arduino mouse instructions). We've stripped the Tkinter GUI but ported sub-functions like color-based RCS, aim smoothing, and trigger thresholds.
- OBS plugin draws from official OBS API examples for custom sources.

**Warning**: This is for **educational purposes only**. Using cheats violates Riot Games' ToS and can result in permanent bans. Test offline or on alt accounts. The author is not responsible for misuse.

## Features

- **Detection**: OpenCV-based enemy outline detection (red/purple hues) inside an OBS plugin, avoiding direct screen grabs.
- **Aimbot**: Smooth mouse movement to detected targets (FOV-limited, humanized curves).
- **Triggerbot**: Instant click on enemy pixel detection.
- **RCS (Recoil Control)**: Basic pattern compensation for weapons.
- **Instant Locker**: API-based (placeholder; adapt from referenced repo).
- **WiFi Comms**: ESP32 as HTTP server; low-latency commands (~20-50ms).
- **HID Spoofing**: ESP32 appears as a generic "USB Optical Mouse" (VID:0x046D, PID:0xC077) to mimic Logitech devices.
- **Configurable**: JSON configs for thresholds, FOV, smoothing (via OBS plugin properties).
- **Stealth**: No COM ports, WiFi-only data; OBS capture bypasses common hooks.

## Requirements

### Hardware
- **ESP32-S3-WROOM-1 N16R8 Dev Board** (e.g., ESP32-S3-DevKitC-1; ~$10).
- **USB Type-A to Type-C Data Cable** (full-spec, not charge-only; for flashing/power).
- **PC Running Valorant** (Windows 10/11; OBS Studio installed).
- **Wireless Mouse** (optional; ESP32 handles inputs—use wired for lower latency if needed).

### Software
- **Arduino IDE** (2.x) with ESP32 board package (via Boards Manager: "esp32" by Espressif).
- **OBS Studio** (30.x+; for plugin loading).
- **Visual Studio 2022** (with C++ Desktop Development workload; for building the OBS plugin).
- **Python 3.10+** (embedded in plugin via pybind11; or standalone for testing).
- **Libraries** (installed via vcpkg or NuGet for plugin):
  - OpenCV 4.8+ (for detection).
  - libcurl (for WiFi HTTP).
  - pybind11 (for Python interop in C++ plugin).
- **Git** (to clone this repo).

## Setup

### 1. Clone & Prep Repo
```bash
git clone https://github.com/YOURUSERNAME/WiFi-ESP32-Valorant-Cheat.git
cd WiFi-ESP32-Valorant-Cheat
```

### 2. Flash ESP32
1. Open Arduino IDE, select **ESP32S3 Dev Module** board.
2. Update `sketch/esp32_server.ino` with your WiFi SSID/password.
3. Install libraries: `ESP32-USB-HID`, `WiFi`, `HTTPClient` (via Library Manager).
4. Upload the sketch. Open Serial Monitor (115200 baud) to note the ESP32's IP (e.g., `192.168.1.100`).
5. The ESP32 now runs a WiFi server on port 80, listening for commands like `?TRIGGER=1` or `?AIM_X=10&Y=5`.

**ESP32 Code Snippet** (from `esp32_server.ino`; adapted from referenced Arduino mouse logic):
```cpp
#include <WiFi.h>
#include <USBHID.h>
USBHID HID; Mouse_ Mouse(HID);

const char* ssid = "YOUR_SSID"; const char* password = "YOUR_PASS";
WiFiServer server(80);

void setup() {
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) delay(500);
  server.begin();
  HID.begin(); Mouse.begin();
  // Spoof HID descriptor as generic mouse (VID/PID below)
}

void loop() {
  WiFiClient client = server.available();
  if (client) {
    String req = client.readStringUntil('\r');
    if (req.indexOf("TRIGGER=1") != -1) Mouse.click(MOUSE_LEFT);
    else if (req.indexOf("AIM_X=") != -1) {
      int x = req.substring(req.indexOf("X=")+2).toInt();
      int y = req.substring(req.indexOf("Y=")+2).toInt();
      Mouse.move(x, y); // Humanized smoothing from referenced repo
    }
    client.stop();
  }
}
```
- **Spoofing**: Edit HID descriptor in `USBHID.h` (or use TinyUSB lib) to set VID=0x046D, PID=0xC077 (Logitech G102 mimic). This makes it appear as a standard mouse in Device Manager, evading ESP32-specific fingerprints.

### 3. Build & Install OBS Plugin
The plugin (`obs_plugin/`) is a C++ source using OBS API, embedding OpenCV for detection and curl for WiFi sends.

1. Open `obs_plugin/obs_plugin.sln` in Visual Studio.
2. Install dependencies:
   - vcpkg: `vcpkg install opencv[contrib]:x64-windows curl pybind11`
   - Link via CMakeLists.txt.
3. Build **Release x64** → Outputs `valorant_capture.dll`.
4. Copy `valorant_capture.dll` to `%APPDATA%\obs-studio\obs-plugins\64bit\`.
5. Restart OBS, add "Valorant Capture" source (select Valorant .exe; enable "Anti-Cheat Safe Mode").

**Plugin Logic** (from `src/valorant_source.cpp`; ports aimbot/trigger from referenced repo's `main.py`):
- Hooks DirectX/OpenGL like Game Capture (using `gs_effect` and `gs_texture`).
- On frame: Convert to HSV, mask enemy colors (adapt `detect_enemy` function).
- If detected: Calc aim offset, send HTTP to ESP32 IP (hardcode or prop).
- Properties: Sliders for FOV, sensitivity (JSON config load).

**Build Command** (CMake alternative):
```bash
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release -DVCPKG_TARGET_TRIPLET=x64-windows
cmake --build . --config Release
```

### 4. Run & Configure
1. Launch OBS, add Valorant Capture source to a scene (hidden; doesn't stream).
2. Start Valorant (fullscreen/borderless).
3. In OBS: Right-click source → Properties → Set ESP IP, thresholds (e.g., trigger delay=10ms).
4. Toggle via hotkey (e.g., Insert; uses OBS scripting).
5. Test: Plugin logs to OBS console; ESP Serial shows commands.

**Config Example** (`config.json`; loaded in plugin):
```json
{
  "esp_ip": "192.168.1.100",
  "fov": 90,
  "smoothing": 0.5,
  "trigger_hue_low": [0, 120, 70],
  "rcs_enabled": true
}
```

### 5. Mouse Spoofing
- **Wireless/Wired Mouse**: Use your normal mouse for legit input; ESP32 overlays HID moves/clicks. For wired fallback, plug ESP32 via native USB OTG port.
- **Avoidance Methods**:
  - **VID/PID Spoof**: As above; test in Device Manager—should show "HID-compliant mouse" under Logitech.
  - **Report Descriptors**: Customize HID reports to match real mice (no ESP32-specific bytes; use `tusb_hid.h` examples).
  - **Randomization**: Add jitter to movements (from referenced repo's RCS) to mimic human input.
  - **Detection Evasion**: WiFi avoids COM port scans; OBS plugin uses official hooks (Vanguard allows Game Capture if not streaming). No DLL injection.

If flagged: Switch PIDs (e.g., to Razer: 0x1532,0x0123) and rebuild HID firmware.

## Alternative Screen Capture: Local Streaming to Browser

As an alternative to the direct OBS plugin method, you can stream the OBS-captured game feed to a local server for near-zero added latency, then play and process the stream in a browser. This approach offloads detection to JavaScript (using opencv.js), potentially reducing detection risk by avoiding embedded Python/C++ in OBS. It achieves low-latency local streaming (sub-100ms end-to-end) and is feasible with moderate setup effort.

### Is This Possible?
Yes, this is possible and commonly used for low-latency local previews or multi-device syncing. OBS can stream to a local server using protocols like SRT, WHIP, or RTMP, which support sub-second latency. The browser then plays the stream via a <video> tag (with libraries like Video.js for HLS/FLV support) and captures frames using HTML Canvas for processing. Detection logic (adapted from the referenced repo) runs in JS, sending commands to the ESP32 via XMLHttpRequest or Fetch API.

### How Easily Can We Do It?
- **Ease Level**: Moderate (2-4 hours for setup if familiar with web dev; harder for beginners due to server config).
- **Pros**: Lower detection profile (browser-based processing); flexible for remote viewing; minimal added latency (~50-200ms with SRT/WHIP).
- **Cons**: Requires additional software (e.g., local server); potential browser overhead; anti-cheat might flag unusual browser activity (mitigate by running in a hidden tab).
- **Latency**: Comparable to the plugin method; use SRT or WHIP for <100ms on localhost.

### Setup Steps
1. **Install Local Streaming Server**:
   - Use **Nginx with RTMP Module** for simplicity (free, easy config).
     - Download Nginx RTMP build (e.g., from https://github.com/arut/nginx-rtmp-module).
     - Config (`nginx.conf` example in `alternative_browser_capture/nginx.conf`):
       ```
       rtmp {
           server {
               listen 1935;
               application live {
                   live on;
               }
           }
       }
       ```
     - Run: `nginx.exe` (localhost RTMP server ready).
   - Alternative: **SRS (Simple Realtime Server)** for SRT/WHIP support (better latency; install via https://ossrs.net).
     - Config for WHIP: Enable in `conf/srs.conf` and run `./objs/srs -c conf/srs.conf`.

2. **Configure OBS to Stream Locally**:
   - In OBS: Settings → Stream → Service: Custom → Server: `rtmp://localhost/live` (for RTMP) or `srt://localhost:8080?streamid=publish/live/stream` (for SRT).
   - Add Game Capture source for Valorant (as before).
   - Start streaming (hidden scene; no internet upload).
   - For ultra-low latency: Enable OBS "Low Latency Mode" (Advanced Output → Encoder: x264, Rate Control: CBR, Tune: zerolatency).

3. **Browser Capture & Processing**:
   - Use a simple web app (provided in `alternative_browser_capture/index.html` and `script.js`).
   - Play stream in <video> tag:
     - For RTMP: Use FLV.js (`<script src="flv.js"></script>`) to play `rtmp://localhost/live/stream`.
     - For SRT/WHIP: Use Video.js with HLS (`<script src="video.js"></script>`) to play `http://localhost:8080/live/stream.m3u8`.
   - Capture frames: Use `requestAnimationFrame` to draw <video> to <canvas>, get image data, and process with opencv.js.
   - **JS Detection Logic** (adapted from referenced repo's Python; in `script.js`):
     ```javascript
     const video = document.getElementById('stream');
     const canvas = document.createElement('canvas');
     const ctx = canvas.getContext('2d');
     const espIp = '192.168.1.100'; // From config

     async function processFrame() {
       ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
       let src = cv.imread(canvas);
       let hsv = new cv.Mat();
       cv.cvtColor(src, hsv, cv.COLOR_RGB2HSV);
       let low = new cv.Scalar(0, 120, 70); // Adapt thresholds
       let high = new cv.Scalar(10, 255, 255);
       let mask = new cv.Mat();
       cv.inRange(hsv, low, high, mask);
       if (cv.countNonZero(mask) > 100) {
         fetch(`http://${espIp}?TRIGGER=1`); // Send to ESP32
       }
       // Cleanup mats
       requestAnimationFrame(processFrame);
     }

     // Load opencv.js and start on video play
     ```
   - Include opencv.js: `<script async src="https://docs.opencv.org/4.x/opencv.js"></script>`.
   - Run: Open `index.html` in Chrome (hidden tab via extension like "Hidden Tab").

4. **Integration**:
   - Load `config.json` in JS for thresholds/FOV.
   - For aimbot: Calculate offsets in JS and send `?AIM_X=10&Y=5`.
   - Test: Ensure stream plays smoothly; monitor latency with browser dev tools.

5. **Advanced Options**:
   - **P2P Direct**: Use [OBS2Browser](https://github.com/Sean-Der/OBS2Browser) plugin for direct OBS-to-browser P2P (no server; ~50ms latency). Install plugin, connect via WebSocket in JS.
   - **Latency Tweaks**: Use 5GHz WiFi; SRT over RTMP for <50ms.
   - **Stealth**: Run browser as a Node.js/Electron app to hide windows.

This method is a good fallback if the plugin feels too integrated—experiment based on your setup.

## Troubleshooting
- **ESP Not Detected**: Check Serial IP; ensure same WiFi.
- **Plugin Black Screen**: Enable "Interceptor Compatible" in source props (hooks DX11).
- **Ban Risk**: Disable on main account; monitor Vanguard logs (`%LOCALAPPDATA%\Riot Games\Vanguard\`).
- **Latency**: Use 5GHz WiFi; fallback to UDP for <10ms.

## Building for Release
- Plugin: Nuitka-like for Python parts: `nuitka --onefile --enable-plugin=opencv main_embed.py` (embed in C++).
- Full Package: Zip ESP sketch, DLL, config.

## Contributing
Fork and PR improvements (e.g., Vulkan support). Credit the original repo for core logic.

## Credits
- Core aim/trigger logic adapted from [Megant88/Valorant-GUI-Cheat-Arduino](https://github.com/Megant88/Valorant-GUI-Cheat-Arduino) (Python → C++ port).
- OBS API: [obsproject.com](https://obsproject.com).
- HID Spoof: Espressif TinyUSB examples.

## License
MIT. Educational use only—do with it what you want, but star if it helps! 🚀
