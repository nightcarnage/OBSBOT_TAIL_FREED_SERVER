# OBSBOT Tail FreeD Camera Simulator

This Python script, `UDP_OBSBOT_TAIL_FREED_SERVER_BETA.py`, simulates a FreeD camera, leveraging an OBSBOT Tail gimbal for baseline orientation and allowing for keyboard adjustments.

## Overview

The script fetches gimbal data (specifically pitch and yaw) from an OBSBOT Tail camera via its HTTP API endpoint: `http://192.168.0.1:27739/obsbot/tail/ai/gimbal`. This data provides the initial orientation.

Keyboard inputs are then used to adjust this orientation:
- **Arrow Keys:** Control pitch and yaw adjustments.
- **A/S Keys:** Control roll adjustments. Roll is exclusively controlled by the keyboard.

The script combines the gimbal data with keyboard inputs and sends the resulting orientation data as FreeD protocol packets over UDP. The destination IP address and port for these packets are configurable, with default values of `127.0.0.1` and `40000` respectively.

## Key Components

### Python Libraries Used:
-   `socket`: For network communication (sending UDP packets).
-   `msvcrt`: For non-blocking keyboard input detection (Windows-specific).
-   `time`: For timing and delays.
-   `threading`: To run the gimbal data polling in a separate thread.
-   `requests`: To fetch data from the OBSBOT Tail camera's HTTP API.

### Core Functions:
-   `pack_24bit(value)`: This function formats numerical values into the 24-bit representation required by the FreeD protocol.
-   `poll_gimbal_data()`: This function runs in a separate thread and continuously fetches the latest pitch and yaw data from the OBSBOT Tail camera.

## Controls

-   **Up Arrow:** Increase Pitch
-   **Down Arrow:** Decrease Pitch
-   **Left Arrow:** Decrease Yaw
-   **Right Arrow:** Increase Yaw
-   **A Key:** Roll Left
-   **S Key:** Roll Right
-   **ESC Key:** Quit the script

## How to Use

1.  **Connect OBSBOT Tail:** Ensure your OBSBOT Tail camera is connected to the same network as the computer running the script and is accessible at `http://192.168.0.1:27739`.
2.  **Run the Script:** Execute `UDP_OBSBOT_TAIL_FREED_SERVER_BETA.py` using a Python interpreter.
    ```bash
    python UDP_OBSBOT_TAIL_FREED_SERVER_BETA.py
    ```
3.  **Receive FreeD Data:** The script will immediately start sending FreeD protocol packets to the configured UDP IP address and port (default: `127.0.0.1:40000`). Ensure your receiving application (e.g., a virtual camera system in Unreal Engine, Aximmetry, etc.) is set up to listen for FreeD data on this address and port.
4.  **Control Orientation:** Use the keyboard controls listed above to adjust the camera's pitch, yaw, and roll.
5.  **Quit:** Press the `ESC` key to stop the script.

---
*Note: This script is currently in a BETA stage.*
