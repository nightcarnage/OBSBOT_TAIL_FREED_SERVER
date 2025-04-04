https://youtu.be/o_Zvlnq0UFI


import socket
import msvcrt
import time
import threading
import requests

# ----------------- Configuration -----------------
UDP_IP = "127.0.0.1"
UDP_PORT = 40000
SEND_RATE_HZ = 60          # target send rate (60 Hz)
ANGLE_SPEED = 60.0         # rotation speed (degrees per second) for held keys
CAMERA_ID = 0              # FreeD camera ID
GIMBAL_URL = "http://192.168.0.1:27739/obsbot/tail/ai/gimbal"

# ----------------- Global Orientation State -----------------
# Baseline orientation from gimbal poll (updated continuously)
baseline_pitch = 0.0   # Will be driven by sensor index 2 (horizontal movement → virtual tilt)
baseline_yaw   = 0.0   # Will be driven by sensor index 1 (vertical movement → virtual pan)
baseline_roll  = 0.0   # Remains zero; roll is controlled via keyboard

# Offset added via keyboard input (accumulated adjustments)
offset_pitch = 0.0
offset_yaw   = 0.0
offset_roll  = 0.0

# Lock to protect baseline orientation updates
orientation_lock = threading.Lock()

# ----------------- Keyboard Input State -----------------
yaw_left_active   = False
yaw_right_active  = False
pitch_up_active   = False
pitch_down_active = False
roll_left_active  = False
roll_right_active = False

last_yaw_left_time   = 0.0
last_yaw_right_time  = 0.0
last_pitch_up_time   = 0.0
last_pitch_down_time = 0.0
last_roll_left_time  = 0.0
last_roll_right_time = 0.0
KEY_HOLD_THRESHOLD   = 0.1  # seconds to consider key released if no repeat

# For printing keyboard events (limited to once every 0.5 sec)
last_keyboard_print_time = 0.0

# ----------------- UDP Socket Setup -----------------
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

# ----------------- Function: pack_24bit -----------------
def pack_24bit(val: int) -> bytes:
    """Convert a signed 32-bit int to 3-byte (24-bit) two’s complement, big-endian."""
    val &= 0xFFFFFFFF
    if val & 0x80000000:
        val = val - 0x100000000
    if val < 0:
        val = (val + (1 << 24)) & 0xFFFFFF
    else:
        val &= 0xFFFFFF
    return val.to_bytes(3, byteorder='big')

# ----------------- Function: send_freed_packet -----------------
def send_freed_packet(pitch_deg: float, yaw_deg: float, roll_deg: float):
    """Construct and send a FreeD packet with the given orientation (degrees)."""
    pitch_int = int(pitch_deg * 32768)
    yaw_int   = int(yaw_deg * 32768)
    roll_int  = int(roll_deg * 32768)
    posX_int = 0
    posY_int = 0
    posZ_int = 0
    zoom_int = 0
    focus_int = 0

    packet = bytearray()
    packet.append(0xD1)                        # Identifier
    packet.append(CAMERA_ID & 0xFF)            # Camera ID
    packet.extend(pack_24bit(pitch_int))       # Pitch
    packet.extend(pack_24bit(yaw_int))         # Yaw
    packet.extend(pack_24bit(roll_int))        # Roll
    packet.extend(pack_24bit(posZ_int))        # Z position
    packet.extend(pack_24bit(posX_int))        # X position
    packet.extend(pack_24bit(posY_int))        # Y position
    packet.extend(pack_24bit(zoom_int))        # Zoom
    packet.extend(pack_24bit(focus_int))       # Focus
    packet.extend(b'\x00\x00')                 # Reserved bytes

    # Compute checksum so that sum(packet) mod 256 equals 64.
    sum_bytes = sum(packet) % 256
    checksum = (64 - sum_bytes) % 256
    packet.append(checksum & 0xFF)

    sock.sendto(packet, (UDP_IP, UDP_PORT))

# ----------------- Thread Function: poll_gimbal_data -----------------
def poll_gimbal_data():
    global baseline_pitch, baseline_yaw, baseline_roll
    while True:
        try:
            response = requests.get(GIMBAL_URL, timeout=2)
            data = response.json()
            degree = data.get("Degree", None)
            if degree and len(degree) >= 3:
                with orientation_lock:
                    # Invert the sensor values:
                    #   - Inverted sensor index 2 drives virtual camera pitch.
                    #   - Inverted sensor index 1 drives virtual camera yaw.
                    baseline_pitch = -degree[2]
                    baseline_yaw   = -degree[1]
                    baseline_roll  = 0.0  # Roll remains keyboard-controlled.
        except Exception:
            pass
        time.sleep(0.1)  # Polling interval; adjust as needed.

# ----------------- Start Gimbal Polling Thread -----------------
gimbal_thread = threading.Thread(target=poll_gimbal_data, daemon=True)
gimbal_thread.start()

# ----------------- Main Loop: Keyboard + FreeD Packet Sending -----------------
print("Starting Combined FreeD Simulator with Gimbal Baseline + Keyboard Offset")
print("  Controls:")
print("    Arrow Up/Down   - Adjust yaw offset (tilt up/down)")
print("    Arrow Left/Right - Adjust pitch offset (pan left/right)")
print("    A / S keys      - Adjust roll offset (roll left/right)")
print("    Esc             - Quit\n")

try:
    last_time = time.time()
    while True:
        # Process keyboard input (non-blocking)
        while msvcrt.kbhit():
            key = msvcrt.getch()
            if key in (b'\x00', b'\xe0'):
                key2 = msvcrt.getch()
                if key2 == b'H':  # Up arrow: decrease yaw offset (pan left)
                    yaw_left_active = True
                    yaw_right_active = False
                    last_yaw_left_time = time.time()
                elif key2 == b'P':  # Down arrow: increase yaw offset (pan right)
                    yaw_right_active = True
                    yaw_left_active = False
                    last_yaw_right_time = time.time()
                elif key2 == b'K':  # Left arrow: decrease pitch offset (tilt up)
                    pitch_up_active = True
                    pitch_down_active = False
                    last_pitch_up_time = time.time()
                elif key2 == b'M':  # Right arrow: increase pitch offset (tilt down)
                    pitch_down_active = True
                    pitch_up_active = False
                    last_pitch_down_time = time.time()
            else:
                if key in (b'a', b'A'):
                    roll_left_active = True
                    roll_right_active = False
                    last_roll_left_time = time.time()
                elif key in (b's', b'S'):
                    roll_right_active = True
                    roll_left_active = False
                    last_roll_right_time = time.time()
                elif key == b'\x1b':  # ESC key
                    print("Exiting simulator.")
                    raise KeyboardInterrupt

        current_time = time.time()
        dt = current_time - last_time
        last_time = current_time

        # Update offsets from keyboard
        if yaw_left_active:
            if current_time - last_yaw_left_time < KEY_HOLD_THRESHOLD:
                offset_yaw -= ANGLE_SPEED * dt
            else:
                yaw_left_active = False
        if yaw_right_active:
            if current_time - last_yaw_right_time < KEY_HOLD_THRESHOLD:
                offset_yaw += ANGLE_SPEED * dt
            else:
                yaw_right_active = False
        if pitch_up_active:
            if current_time - last_pitch_up_time < KEY_HOLD_THRESHOLD:
                offset_pitch -= ANGLE_SPEED * dt
            else:
                pitch_up_active = False
        if pitch_down_active:
            if current_time - last_pitch_down_time < KEY_HOLD_THRESHOLD:
                offset_pitch += ANGLE_SPEED * dt
            else:
                pitch_down_active = False
        if roll_left_active:
            if current_time - last_roll_left_time < KEY_HOLD_THRESHOLD:
                offset_roll -= ANGLE_SPEED * dt
            else:
                roll_left_active = False
        if roll_right_active:
            if current_time - last_roll_right_time < KEY_HOLD_THRESHOLD:
                offset_roll += ANGLE_SPEED * dt
            else:
                roll_right_active = False

        # Optionally print keyboard offsets (but not the baseline)
        if current_time - last_keyboard_print_time > 0.5:
            if (yaw_left_active or yaw_right_active or
                pitch_up_active or pitch_down_active or
                roll_left_active or roll_right_active):
                print(f"Keyboard Offset -> Pitch: {offset_pitch:.2f}°, Yaw: {offset_yaw:.2f}°, Roll: {offset_roll:.2f}°")
                last_keyboard_print_time = current_time

        # Combine baseline and keyboard offsets
        with orientation_lock:
            out_pitch = baseline_pitch + offset_pitch
            out_yaw   = baseline_yaw   + offset_yaw
            out_roll  = baseline_roll  + offset_roll

        # Wrap yaw to [-180, 180] if necessary
        if out_yaw > 180.0:
            out_yaw -= 360.0
        elif out_yaw < -180.0:
            out_yaw += 360.0

        send_freed_packet(out_pitch, out_yaw, out_roll)
        time.sleep(max(0, (1.0 / SEND_RATE_HZ) - (time.time() - current_time)))
except KeyboardInterrupt:
    pass
finally:
    sock.close()
