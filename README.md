# OBSBOT Tail FreeD Server (Beta)

https://youtu.be/o_Zvlnq0UFI

This project provides a specialized **UDP bridge** for the **OBSBOT Tail** camera for use with Unreal Engine.

Unlike standard PTZ controllers that *drive* the camera’s movement, this server acts as a **FreeD data provider and movement offset manager**.

It is designed to run **alongside AI tracking or other control scripts** (such as `nightcarnage/OBSBOT_TAIL_PTZ_IP`) to:

- Provide real-time positioning data  
- Allow fine-tuned framing adjustments  
- Avoid breaking the camera’s internal tracking logic  

---

## 🎯 Purpose

### Movement Offsetting
Apply incremental offsets to the Virtual camera’s current **Pan, Tilt, and Roll**.  

### FreeD Protocol Bridge
Translates the camera’s internal positioning into **industry-standard FreeD**, enabling compatibility with:
- VR / AR pipelines
- Unreal Engine
- OBS and broadcast tooling

### Data Aggregation
Continuously polls the camera’s API to maintain a local **source of truth** for gimbal state, then serves it over UDP.

---

## 🛠 Features

- **UDP Command Listener**  
  Listens for incoming JSON/UDP packets to trigger movement offsets.

- **FreeD Telemetry Output**  
  Streams Pan / Tilt / Roll / Zoom over UDP in FreeD format.

- **Beta Offset Logic**  
  Experimental logic for applying *relative offsets* using OBSBOT’s absolute-position API.

- **Automatic Polling**  
  Keeps gimbal state synced with camera hardware every few milliseconds.

---

## ⌨️ Controls & Offsets

This script does **not** include a keyboard camera control UI  
(use the nightcarnage/OBSBOT_TAIL_PTZ_IP script for manual control).

It listens for the following **offset commands**:

| Action       | Description                                      |
|-------------|--------------------------------------------------|
| Tilt Offset | Adjusts vertical framing (up / down)             |
| Pan Offset  | Adjusts horizontal framing (left / right)        |
| Roll Offset | Fine-tunes horizon leveling                      |
| FreeD Out   | Continuous stream of D1 hex packets              |

