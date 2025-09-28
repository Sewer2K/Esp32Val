# WiFi-ESP32-Valorant-Cheat

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## Overview

This repository serves as a proof-of-concept (PoC) for an external Valorant cheat utilizing an [ESP32 S3 Dev Board - 16MB Flash, 8MB PSRAM](https://www.robotics.org.za/ESP32-S3-N16R8-EXT) to implement hardware-based aimbot and triggerbot functionality over WiFi. This approach diverges from traditional Arduino setups by leveraging the ESP32's built-in WiFi capabilities, which helps minimize cable usage and potential detection risks.

To circumvent Valorant's anti-cheat system (Vanguard), which flags direct screen captures (e.g., via screen capture libraries), the PoC employs a custom OBS plugin designed to emulate the official Game Capture Source. This plugin safely captures the Valorant window by leveraging OBS's graphics hooks for DirectX/OpenGL interception. It processes the video feed using OpenCV-like techniques to detect enemy colors or body outlines and transmits trigger or aim commands to the ESP32 via HTTP over a local WiFi network.

The ESP32 emulates a generic USB HID mouse to handle inputs such as clicks and movements, with its identity spoofed to resemble a standard device, thereby reducing the likelihood of hardware fingerprinting detection. This setup draws inspiration from external color-based cheat designs but emphasizes stealth by avoiding memory access, direct PC-side inputs, and integrating with OBS for a more legitimate capture method.

**Key Inspirations & Code Reuse**:
- The aimbot and triggerbot logic is heavily influenced by [Megant88/Valorant-GUI-Cheat-Arduino](https://github.com/Megant88/Valorant-GUI-Cheat-Arduino), adapting elements from their Python `main.py` and Arduino mouse instructions. The Tkinter GUI has been removed, but sub-functions like color-based recoil control system (RCS), aim smoothing, and trigger thresholds have been incorporated.
- The OBS plugin concept is derived from examples provided by the official OBS API.

**Warning**: This PoC is intended for **educational purposes only**. The use of cheats violates Riot Games' Terms of Service and may lead to permanent bans. Testing should be conducted offline or on alternate accounts. The author bears no responsibility for any misuse.

## Features

- **Detection**: Utilizes OpenCV-based enemy outline detection (focusing on red/purple hues) within the OBS plugin, avoiding direct screen capture methods.
- **Aimbot**: Provides smooth mouse movement towards detected targets, limited by a field-of-view (FOV) setting and incorporating humanized movement curves.
- **Triggerbot**: Executes instant clicks upon detecting enemy pixels.
- **RCS (Recoil Control)**: Applies basic pattern compensation for weapon recoil.
- **Instant Locker**: Includes an API-based feature (placeholder, adaptable from the referenced repo).
- **WiFi Communications**: The ESP32 operates as an HTTP server, facilitating low-latency command transmission (~20-50ms).
- **HID Spoofing**: Configures the ESP32 to mimic a generic "USB Optical Mouse" (e.g., VID:0x046D, PID:0xC077) to resemble Logitech devices.
- **Configurable**: Supports JSON configuration files for adjusting thresholds, FOV, and smoothing via OBS plugin properties.
- **Stealth**: Employs WiFi-only data transfer with no COM ports, and the OBS capture method bypasses common anti-cheat hooks.

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
- Clone the repository using Git and navigate to the project directory.

### 2. Flash ESP32
- Configure the Arduino IDE with the ESP32S3 Dev Module board.
- Update the WiFi settings in the configuration file with your network SSID and password.
- Install necessary libraries through the Library Manager.
- Upload the configuration to the ESP32 and check the Serial Monitor for the device's IP address.
- The ESP32 will then function as a WiFi server, ready to receive commands such as trigger or aim instructions.

### 3. Build & Install OBS Plugin
- Open the plugin solution file in Visual Studio.
- Install the required dependencies using vcpkg or NuGet as specified.
- Build the plugin in Release x64 mode to generate the dynamic link library (DLL) file.
- Install the DLL into the OBS plugins directory and restart OBS to add the "Valorant Capture" source.

### 4. Run & Configure
- Launch OBS and add the Valorant Capture source to a scene, keeping it hidden from streams.
- Start Valorant in fullscreen or borderless mode.
- Access the source properties in OBS to configure the ESP IP and set thresholds.
- Use a designated hotkey to toggle the feature and test the setup.

### 5. Mouse Spoofing
- Utilize your existing mouse for legitimate inputs, with the ESP32 overlaying additional HID moves and clicks. A wired connection can be used for reduced latency if necessary.
- Implement spoofing techniques to disguise the ESP32 as a standard mouse, adjusting device identifiers and randomizing movements to mimic human behavior.

## Alternative Screen Capture: Local Streaming to Browser

As an alternative to the direct OBS plugin approach, this PoC explores streaming the OBS-captured game feed to a local server to achieve near-zero added latency, then playing and processing the stream within a browser. This method offloads detection to JavaScript using opencv.js, potentially lowering the detection risk by avoiding embedded Python or C++ within OBS. The setup aims for a low-latency local streaming experience (sub-100ms end-to-end) and is feasible with a moderate level of effort.

### Is This Possible?
Yes, this is achievable and aligns with techniques used for low-latency local previews or multi-device synchronization. OBS can transmit the game feed to a local server using protocols such as SRT, WHIP, or RTMP, which are designed to support sub-second latency. The browser plays this stream through a <video> element, supported by libraries like Video.js for HLS or FLV playback, and captures frames using HTML Canvas. Detection logic, inspired by the referenced repo, is executed in JavaScript, with commands sent to the ESP32 via standard web requests.

### How Easily Can We Do It?
- **Ease Level**: Moderate (2-4 hours for setup if familiar with web development; more challenging for beginners due to server configuration requirements).
- **Pros**: Offers a lower detection profile through browser-based processing, provides flexibility for remote viewing, and introduces minimal additional latency (~50-200ms with SRT/WHIP).
- **Cons**: Requires additional software (e.g., a local server), may introduce browser overhead, and anti-cheat systems might flag unusual browser activity (mitigated by running in a hidden tab).
- **Latency**: Comparable to the plugin method, with SRT or WHIP enabling latency below 100ms on localhost.

### Setup Steps
1. **Install Local Streaming Server**:
   - Opt for **Nginx with RTMP Module** for a straightforward setup (free and easily configurable).
   - Alternatively, consider **SRS (Simple Realtime Server)** for SRT or WHIP support, which offers improved latency.

2. **Configure OBS to Stream Locally**:
   - Adjust OBS stream settings to use a custom server address (e.g., `rtmp://localhost/live` for RTMP or `srt://localhost:8080` for SRT).
   - Incorporate a Game Capture source for Valorant and initiate streaming within a hidden scene, ensuring no internet upload occurs.
   - Enhance performance with OBS's Low Latency Mode for reduced delay.

3. **Browser Capture & Processing**:
   - Utilize a basic web application to play the stream in a <video> tag, supporting RTMP with FLV.js or SRT/WHIP with Video.js and HLS.
   - Capture frames by drawing the video to a <canvas> element and processing them with opencv.js.
   - Implement detection logic in JavaScript to identify enemies and send corresponding commands to the ESP32.

4. **Integration**:
   - Load configuration settings from a JSON file within the JavaScript environment to adjust thresholds and FOV.
   - Calculate aim offsets in the browser and transmit them to the ESP32.
   - Verify the stream's performance and monitor latency using browser developer tools.

5. **Advanced Options**:
   - Explore **P2P Direct** using tools like OBS2Browser for a serverless connection, potentially reducing latency further.
   - Optimize latency with 5GHz WiFi and prefer SRT over RTMP.
   - Enhance stealth by running the browser within a Node.js or Electron application to conceal windows.

This alternative method serves as a viable backup if the plugin approach feels overly integrated, allowing for experimentation based on your specific setup.

## Troubleshooting
- **ESP Not Detected**: Verify the Serial IP and ensure both devices are on the same WiFi network.
- **Plugin Black Screen**: Activate "Interceptor Compatible" in the source properties to ensure proper DirectX hooking.
- **Ban Risk**: Deactivate on your main account and keep an eye on Vanguard logs.
- **Latency**: Switch to 5GHz WiFi or consider UDP for reduced delay.

## Building for Release
- Prepare the plugin for release using a Nuitka-like process for embedded Python components.
- Package the complete solution including the ESP configuration, DLL, and configuration files.

## Contributing
- Fork the repository and submit pull requests for enhancements, such as Vulkan support. Acknowledge the original repo for its foundational logic.

## Credits
- Core aim and trigger logic adapted from [Megant88/Valorant-GUI-Cheat-Arduino](https://github.com/Megant88/Valorant-GUI-Cheat-Arduino).
- OBS API inspiration from [obsproject.com](https://obsproject.com).
- HID Spoof techniques derived from Espressif TinyUSB examples.

## License
MIT. Intended for educational use only—feel free to explore, but please star the repo if it proves helpful! 🚀
