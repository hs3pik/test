# 📖 คู่มือการสร้าง Mumble Radio Gateway บน Raspberry Pi (Edition: จอ LCD 16x2)

Mumble Gateway คือระบบที่ทำหน้าที่เป็น "สะพาน" เชื่อมต่อระหว่างแอปพลิเคชันสนทนาด้วยเสียงผ่านอินเทอร์เน็ต (Mumble) กับ เครือข่ายวิทยุสื่อสาร (RF) เวอร์ชันนี้มาพร้อมกับ Web Dashboard สำหรับดูสถานะ, เปลี่ยนห้อง, สั่งรีสตาร์ทระบบผ่านมือถือ และมีระบบป้องกันคีย์ค้าง (Watchdog) ในตัว

💡 **ทริคก่อนเริ่มทำ:** 
ในคู่มือนี้จะมีการอ้างอิงถึงตำแหน่งไฟล์ หากไม่แน่ใจว่าเครื่อง Raspberry Pi ของคุณใช้ชื่อ User ว่าอะไร ให้พิมพ์คำสั่ง `whoami` ใน Terminal ให้นำชื่อที่ได้ไปแทนที่คำว่า `[USERNAME]` ในขั้นตอนการตั้งค่า Service ทุกจุดนะครับ

## บทที่ 1: อุปกรณ์ที่ต้องเตรียม (Hardware & Components)

* **คอมพิวเตอร์บอร์ดเดี่ยว:** Raspberry Pi  พร้อม MicroSD Card และอะแดปเตอร์ (ติดตั้ง Raspberry Pi OS Lite 32-bit รุ่น Bookworm)
* **ระบบเสียง:** USB Sound Card (แนะนำชิป CM108 / C-Media ซึ่งเป็นมาตรฐานยอดนิยม) และสาย Audio
* **วงจรควบคุมและสั่งงาน:**
  * IC Optocoupler เบอร์ PC817 (สำคัญมาก! เพื่อแยกกราวด์ป้องกันบอร์ด Pi พัง)
  * ตัวต้านทาน (Resistor) 1k Ohm และ 10k Ohm
  * สวิตช์ปุ่มกด (Push Button) 2 ตัว สำหรับกดเปลี่ยนห้องหน้าเครื่อง
* **หน้าจอแสดงผล:** จอ LCD 16x2 พร้อมโมดูล I2C
* **อุปกรณ์สื่อสาร:** วิทยุสื่อสาร พร้อมสายนำสัญญาณและเสาอากาศ

## บทที่ 2: การเตรียมระบบปฏิบัติการและการตั้งค่าเครือข่าย

แนะนำให้ใช้ Raspberry Pi OS Lite (32-bit) เวอร์ชัน Bookworm ขึ้นไป

**1. อัปเดตเวลาให้ตรงกับประเทศไทย (สำคัญต่อระบบ Mumble)**
พิมพ์คำสั่ง:
```bash
sudo timedatectl set-timezone Asia/Bangkok
sudo systemctl restart systemd-timesyncd
```

**2. ตั้งค่า Wi-Fi (หากนำไปติดตั้งโดยไม่ใช้สาย LAN)**
พิมพ์คำสั่ง:
```bash
sudo raspi-config
```
* ไปที่ `System Options` -> เลือก `Wireless LAN`
* ใส่รหัสประเทศ (เลือก `TH Thailand`)
* ใส่ชื่อ SSID (ชื่อ Wi-Fi) และรหัสผ่าน
* ยังไม่ต้องรีบูต ให้ไปทำข้อ 3 ต่อ

**3. เปิดใช้งาน I2C สำหรับหน้าจอ LCD**
* ในหน้าจอ `raspi-config` ไปที่ `Interface Options` -> เลือก `I2C` -> กด `Yes`
* เลือก `Finish` และรีบูตเครื่องด้วยคำสั่ง: 
```bash
sudo reboot
```

## บทที่ 3: การต่อวงจรฮาร์ดแวร์ (Wiring Diagram)

**1. การต่อจอ LCD (แบบ I2C)**
จอ LCD ที่มีโมดูล I2C ด้านหลังจะมีขั้วต่อ 4 ขา ให้เสียบสายจัมเปอร์เข้ากับบอร์ด Raspberry Pi ตามนี้ครับ:
* `VCC` ต่อเข้าขา Pin 2 หรือ 4 (ไฟ 5V)
* `GND` ต่อเข้าขา Pin 6 (Ground)
* `SDA` ต่อเข้าขา Pin 3 (GPIO 2)
* `SCL` ต่อเข้าขา Pin 5 (GPIO 3)

**2. วงจรแยกกราวด์ PTT ด้วย Optocoupler (PC817)**
วงจรนี้สำคัญมาก! ตัว PC817 จะเป็นตัวกลางคั่นระหว่าง "ฝั่งคอมพิวเตอร์ (Pi)" และ "ฝั่งวิทยุสื่อสาร" เพื่อป้องกันไฟกระชากหรือสัญญาณกวนกัน
```text
[ Raspberry Pi ]                [ IC: PC817 ]               [ วิทยุสื่อสาร ]
[Pin 37] GPIO 26 -----> [R 1k] ---> (ขา 1)     (ขา 4) ------------> ขั้วสาย PTT (วิทยุ)
[Pin 39] Ground  -----------------> (ขา 2)     (ขา 3) ------------> ขั้วสาย Ground (วิทยุ)
(คำแนะนำ: ขา 1 ของตัว PC817 จะอยู่ฝั่งที่มีรอยบุ๋ม หรือจุดกลมๆ เล็กๆ บนตัวถังพลาสติก)
```

**3. การต่อปุ่มกด (Push Button) เปลี่ยนห้อง**
ไม่ต้องใช้ตัวต้านทานต่อภายนอก เพราะเราเขียนโค้ดเปิดใช้โหมด Internal Pull-Up ไว้ใน Raspberry Pi แล้ว ให้ต่อสายตรงๆ ได้เลยครับ
```text
[ ปุ่มที่ 1 : UP (เลื่อนขึ้น) ]
ขาด้านที่ 1 ต่อ Pin 11 (GPIO 17)
ขาด้านที่ 2 ต่อ Pin  9 (Ground)

[ ปุ่มที่ 2 : DOWN (เลื่อนลง) ]
ขาด้านที่ 1 ต่อ Pin 13 (GPIO 27)
ขาด้านที่ 2 ต่อ Pin 14 (Ground)
```

**4. การต่อระบบเสียง (USB Sound Card)**
```text
[ USB Sound Card ]                                [ วิทยุสื่อสาร ]
ช่อง หูฟัง (สีเขียว/สัญลักษณ์หูฟัง) ----------------> ช่องเสียบ ไมโครโฟน (Mic In) ของวิทยุ
ช่อง ไมค์  (สีชมพู/สัญลักษณ์ไมค์)   ----------------> ช่องเสียบ หูฟัง/ลำโพง (Spk Out) ของวิทยุ
```

## บทที่ 4: การติดตั้งโปรแกรมและไลบรารี

เพื่อป้องกัน Error บนระบบปฏิบัติการ OS รุ่นใหม่ ให้พิมพ์คำสั่งตามนี้ทีละบรรทัดครับ:

**1. ติดตั้งเครื่องมือพื้นฐานและไลบรารีระบบ (ต้องทำก่อนเสมอ)**

```bash

sudo apt update
sudo apt install -y python3-pip python3-dev build-essential i2c-tools portaudio19-dev libasound2-dev libopenjp2-7 libtiff6 libopenblas0
```

**2. ติดตั้งไลบรารี Python ทั้งหมดที่ระบบต้องการ:**

```bash
sudo pip3 install pymumble smbus2 pyaudio numpy flask luma.oled pillow rpi-lgpio --break-system-packages
```

## บทที่ 5: การรันระบบ Gateway หลัก (พร้อมระบบ WebUI)

⚠️ **หมายเหตุสำคัญก่อนเริ่มรันโปรแกรม:**
* **ต้องต่อฮาร์ดแวร์ก่อน:** กรุณาเสียบ USB Sound Card และต่อสายจอ LCD I2C เข้ากับบอร์ด Pi ให้เรียบร้อย หากไม่ต่ออุปกรณ์ โปรแกรมจะเช็คฮาร์ดแวร์ไม่ผ่านและฟ้อง Error ทันที
* **เรื่อง USB Sound Card:** โค้ดที่ออกแบบมานี้ ตั้งค่าเริ่มต้นให้ค้นหา Sound Card ชิป `CM108 (C-Media)` หากคุณใช้ Sound Card ยี่ห้อ/รุ่นอื่น คุณจะต้องเข้าไปตรวจสอบชื่อ (ด้วยคำสั่ง `lsusb` หรือ `aplay -l`) แล้วนำชื่อนั้นมาแก้แทนคำว่า "C-Media" ในโค้ด

สร้างไฟล์โปรแกรมหลัก:
```bash
nano ~/gateway.py
```

คัดลอกโค้ดด้านล่างไปวาง (🔴 ห้ามลืมเปลี่ยนค่าในส่วน CONFIGURATION ให้ตรงกับระบบของคุณ):

```python
import time
import threading
import RPi.GPIO as GPIO
import pyaudio
import numpy as np
from flask import Flask, jsonify, render_template_string, request
import logging
import socket
import sys
import subprocess
import ssl
import json
import os
from pymumble_py3 import Mumble

# นำเข้าไลบรารีจอ OLED
from luma.core.interface.serial import i2c
from luma.oled.device import sh1106
from luma.core.render import canvas
from PIL import ImageFont

# ปิด Log ของ Flask ไม่ให้รก Terminal
log = logging.getLogger('werkzeug')
log.setLevel(logging.ERROR)

# =======================================================
# แก้ปัญหา SSL สำหรับ Python 3.13 ขึ้นไป
if not hasattr(ssl, 'wrap_socket'):
    def _wrap_socket(sock, keyfile=None, certfile=None, server_side=False, cert_reqs=ssl.CERT_NONE, ssl_version=ssl.PROTOCOL_TLS, ca_certs=None, do_handshake_on_connect=True, suppress_ragged_eofs=True, ciphers=None):
        context = ssl.SSLContext(ssl_version)
        context.verify_mode = cert_reqs
        context.check_hostname = False
        if certfile:
            context.load_cert_chain(certfile, keyfile)
        return context.wrap_socket(sock, server_side=server_side, do_handshake_on_connect=do_handshake_on_connect, suppress_ragged_eofs=suppress_ragged_eofs)
    ssl.wrap_socket = _wrap_socket
# =======================================================

# ==================== 🔴 CONFIGURATION SYSTEM ====================
CONFIG_FILE = "/home/mumble/config.json"

DEFAULT_CONFIG = {
    "SERVER_IP": "192.168.10.20",
    "PORT": 64738,
    "PASSWORD": "your_password",
    "USERNAME": "Mumble-Gateway",
    "ROOM_1": "CH1", "ROOM_1_PASS": "",
    "ROOM_2": "CH2", "ROOM_2_PASS": "",
    "ROOM_3": "CH3", "ROOM_3_PASS": "",
    "ROOM_4": "CH4", "ROOM_4_PASS": "",
    "ROOM_5": "CH5", "ROOM_5_PASS": "",
    "ROOM_6": "CH6", "ROOM_6_PASS": "",
    "VOX_THRESHOLD": 400,
    "VOX_HANG_TIME": 0.8,
    "TX_HANG_TIME": 0.7
}

def load_config():
    if not os.path.exists(CONFIG_FILE):
        with open(CONFIG_FILE, 'w') as f:
            json.dump(DEFAULT_CONFIG, f, indent=4)
        return DEFAULT_CONFIG
    try:
        with open(CONFIG_FILE, 'r') as f:
            return json.load(f)
    except Exception as e:
        print(f"Error loading config: {e}")
        return DEFAULT_CONFIG

cfg = load_config()

SERVER_IP = cfg.get("SERVER_IP", "")
PORT = int(cfg.get("PORT", 64738))
PASSWORD = cfg.get("PASSWORD", "")
USERNAME = cfg.get("USERNAME", "Mumble-Gateway")

# โหลดห้อง 1-6 และรหัสผ่าน
ROOMS = []
ROOM_PASSWORDS = []
for i in range(1, 7):
    room_name = str(cfg.get(f"ROOM_{i}", "")).strip()
    room_pass = str(cfg.get(f"ROOM_{i}_PASS", "")).strip()
    if room_name:
        ROOMS.append(room_name)
        if room_pass:
            ROOM_PASSWORDS.append(room_pass)

if not ROOMS:
    ROOMS = ["Root"] 

ROOM_PASSWORDS = list(set(ROOM_PASSWORDS))

VOX_THRESHOLD = float(cfg.get("VOX_THRESHOLD", 400))
VOX_HANG_TIME = float(cfg.get("VOX_HANG_TIME", 0.8))
TX_HANG_TIME = float(cfg.get("TX_HANG_TIME", 0.7))

OLED_ADDRESS = 0x3C  # 🔴 Address ของจอ OLED

current_room_idx = 0  
ROOM = ROOMS[current_room_idx]

# --- ตั้งค่า Hardware ---
GPIO_PTT = 26  
GPIO_BTN_UP = 17    
GPIO_BTN_DOWN = 27  

FORMAT = pyaudio.paInt16
CHANNELS = 1
RATE = 48000
CHUNK = 960  
# =======================================================

# --- Safe OLED Initialization & Splash Screen ---
device = None
oled_enabled = False
lcd_lock = threading.Lock()
font = None

try:
    serial = i2c(port=1, address=OLED_ADDRESS)
    device = sh1106(serial)  # ใช้ Driver SH1106
    oled_enabled = True
    print(f"✅ OLED 1.3\" Connected at Address 0x{OLED_ADDRESS:02X}")
    
    try: font = ImageFont.load_default()
    except: pass
    
except Exception as e:
    print(f"⚠️ Warning: Could not initialize OLED ({e})")

def update_display(line1, line2):
    if not oled_enabled or device is None:
        return
    with lcd_lock:
        try:
            with canvas(device) as draw:
                # วาดกรอบสี่เหลี่ยมแต่งหน้าจอ
                draw.rectangle(device.bounding_box, outline="white", fill="black")
                # แสดงข้อความ
                draw.text((8, 12), str(line1), fill="white", font=font)
                draw.text((8, 38), str(line2), fill="white", font=font)
        except Exception as e:
            pass

# 🟢 Boot Splash Screen 
if oled_enabled:
    # 1. แสดงค้าง 1 วินาที (เว้นวรรคให้ข้อความอยู่ตรงกลางจอ)
    update_display("  Dx Solution", "    HS3PIK")
    time.sleep(1)
    
    # 2. ดับแว๊บนึง แล้วกระพริบ 1 ครั้ง
    device.clear()
    time.sleep(0.3)
    update_display("  Dx Solution", "    HS3PIK")
    time.sleep(0.5)
    
    # 3. โชว์ Mumble Gateway
    update_display(" Mumble Gateway", "")
    time.sleep(1)
    device.clear()

# ตั้งค่า GPIO
GPIO.setwarnings(False)
GPIO.setmode(GPIO.BCM)
GPIO.setup(GPIO_PTT, GPIO.OUT, initial=GPIO.LOW)
GPIO.setup(GPIO_BTN_UP, GPIO.IN, pull_up_down=GPIO.PUD_UP)
GPIO.setup(GPIO_BTN_DOWN, GPIO.IN, pull_up_down=GPIO.PUD_UP)

# ==================== BUTTON CALLBACKS ====================
def change_room(direction, source="Button"):
    global current_room_idx, ROOM, mumble
    if not ROOMS: return
    
    if direction == "UP":
        current_room_idx = (current_room_idx + 1) % len(ROOMS)
    elif direction == "DOWN":
        current_room_idx = (current_room_idx - 1) % len(ROOMS)
        
    ROOM = ROOMS[current_room_idx]
    
    if 'mumble' in globals() and mumble and mumble.is_alive():
        target_channel = mumble.channels.find_by_name(ROOM)
        if target_channel:
            target_channel.move_in()
            msg = "Switched!" if source == "Button" else "Switched(Web)!"
            update_display(f"Room: {ROOM}", msg)
            print(f"🔄 Switched to room: {ROOM}")

def btn_up_callback(channel): change_room("UP")
def btn_down_callback(channel): change_room("DOWN")

GPIO.add_event_detect(GPIO_BTN_UP, GPIO.FALLING, callback=btn_up_callback, bouncetime=500)
GPIO.add_event_detect(GPIO_BTN_DOWN, GPIO.FALLING, callback=btn_down_callback, bouncetime=500)

# ==================== SYSTEM CHECKS ====================
p = None
stream_out = None
stream_in = None

def check_audio_device():
    global p, stream_out, stream_in
    update_display("System Check", "Audio (USB)")
    time.sleep(1.5) 
    
    while True:
        p = pyaudio.PyAudio()
        usb_idx = None
        for i in range(p.get_device_count()):
            dev_info = p.get_device_info_by_index(i)
            if "USB" in dev_info['name'] or "C-Media" in dev_info['name']:
                usb_idx = i
                break
                
        if usb_idx is not None:
            update_display("Audio Status", "USB Card OK!")
            time.sleep(1) 
            break
        else:
            p.terminate() 
            update_display("Error: Audio", "No USB Card")
            time.sleep(2)
            
    stream_out = p.open(format=FORMAT, channels=CHANNELS, rate=RATE, output=True, output_device_index=usb_idx)
    stream_in = p.open(format=FORMAT, channels=CHANNELS, rate=RATE, input=True, frames_per_buffer=CHUNK, input_device_index=usb_idx)

def get_network_info():
    try:
        route_output = subprocess.check_output("ip route get 8.8.8.8", shell=True).decode()
        net_type = "WiFi" if "wlan" in route_output else "LAN" if "eth" in route_output else "Net"
        s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        s.connect(("8.8.8.8", 80))
        ip = s.getsockname()[0]
        s.close()
        return net_type, ip
    except Exception:
        return "Unknown", "No IP"

def wait_for_network():
    update_display("System Check", "Network")
    while True:
        try:
            net_type, ip_address = get_network_info()
            if ip_address != "No IP":
                update_display(f"Net: {net_type}", f"{ip_address}")
                time.sleep(1.5) 
                break
        except Exception:
            pass
        update_display("Network Error", "Waiting IP...")
        time.sleep(2)

# ==================== VARIABLES ====================
last_audio_time = 0
is_transmitting = False
is_vox_active = False  
last_idle_text = ""    
user_speaking_status = {}

# ==================== FUNCTIONS ====================
def sound_received_handler(user, soundchunk):
    global last_audio_time
    last_audio_time = time.time()
    if user and 'name' in user:
        user_speaking_status[user['name']] = time.time()
    try:
        if stream_out and stream_out.is_active():
            stream_out.write(soundchunk.pcm)
    except Exception:
        pass

def vox_monitor_thread(mumble_instance):
    global is_vox_active, is_transmitting
    last_vox_trigger_time = 0

    while True:
        try:
            if stream_in and stream_in.is_active():
                data = stream_in.read(CHUNK, exception_on_overflow=False)
                if is_transmitting:
                    continue

                audio_data = np.frombuffer(data, dtype=np.int16)
                rms = np.sqrt(np.mean(np.square(audio_data.astype(np.float32))))

                if rms > VOX_THRESHOLD:
                    last_vox_trigger_time = time.time()
                    is_vox_active = True
                    mumble_instance.sound_output.add_sound(data)
                else:
                    if time.time() - last_vox_trigger_time > VOX_HANG_TIME:
                        is_vox_active = False
        except Exception:
            time.sleep(0.01)

# ==================== WEB UI (FLASK) ====================
app = Flask(__name__)

HTML_TEMPLATE = """
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Gateway Monitor & Config</title>
    <style>
        :root { --bg-dark: #0f151c; --bg-panel: #1a2332; --bg-inner: #111827; --text-main: #e2e8f0; --text-cyan: #00e5ff; --border-color: #2d3748; --color-ready: #22c55e; --color-tx: #ef4444; --color-rx: #3b82f6; }
        body { font-family: 'Segoe UI', sans-serif; background-color: var(--bg-dark); color: var(--text-main); margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }
        .container { background-color: var(--bg-panel); border: 1px solid var(--border-color); border-radius: 12px; padding: 20px 30px; width: 100%; max-width: 1000px; position: relative; box-shadow: 0 10px 25px rgba(0,0,0,0.5);}
        .header { display: flex; justify-content: space-between; border-bottom: 1px solid var(--border-color); padding-bottom: 15px; margin-bottom: 25px; align-items: center;}
        .header h1 { color: var(--text-cyan); margin: 0; font-size: 1.8rem; }
        
        .btn-group { display: flex; gap: 10px; }
        .btn-action { background-color: var(--bg-inner); border: 1px solid var(--border-color); color: var(--text-main); padding: 8px 15px; border-radius: 8px; cursor: pointer; transition: 0.3s; }
        .btn-restart { border-color: #ef4444; color: #ef4444; }
        .btn-restart:hover { background-color: rgba(239, 68, 68, 0.2); }
        .btn-action:hover { background-color: rgba(255, 255, 255, 0.1); }

        .grid-layout { display: grid; grid-template-columns: 300px 1fr; gap: 20px; }
        .sidebar { background-color: var(--bg-inner); border: 1px solid var(--border-color); border-radius: 8px; padding: 15px; display: flex; flex-direction: column; max-height: 400px; }
        .user-list { list-style: none; padding: 0; margin: 0; overflow-y: auto; flex-grow: 1; }
        .user-item { background-color: var(--bg-panel); padding: 10px 15px; margin-bottom: 8px; border-radius: 6px; display: flex; justify-content: space-between; }
        .mic-icon { width: 12px; height: 12px; border-radius: 50%; background-color: #4a5568; }
        .mic-active { background-color: var(--color-ready); box-shadow: 0 0 8px var(--color-ready); }
        
        .main-panel { display: flex; flex-direction: column; gap: 15px; }
        .top-status { background-color: var(--bg-inner); border: 1px solid var(--border-color); border-radius: 8px; padding: 15px 20px; display: flex; justify-content: space-between; align-items: center; }
        .room-controls { display: flex; align-items: center; gap: 15px; }
        .btn-room { background-color: var(--bg-panel); border: 1px solid var(--border-color); color: var(--text-cyan); font-size: 1.2rem; padding: 5px 15px; border-radius: 8px; cursor: pointer; }
        .display-area { background-color: #0b111a; border: 1px dashed var(--border-color); border-radius: 8px; flex-grow: 1; display: flex; justify-content: center; align-items: center; min-height: 200px; }
        .status-badge { font-size: 1.8rem; font-weight: bold; padding: 15px 40px; border-radius: 50px; letter-spacing: 2px; text-align: center; }
        .bg-ready { color: var(--color-ready); border: 2px solid var(--color-ready); background: rgba(34, 197, 94, 0.1); }
        .bg-tx { color: var(--color-tx); border: 2px solid var(--color-tx); background: rgba(239, 68, 68, 0.1); }
        .bg-rx { color: var(--color-rx); border: 2px solid var(--color-rx); background: rgba(59, 130, 246, 0.1); }
        .bg-err { color: #facc15; border: 2px solid #facc15; background: rgba(250, 204, 21, 0.1); font-size: 1.2rem; }
        
        .footer { text-align: right; margin-top: 20px; padding-top: 15px; border-top: 1px solid var(--border-color); font-size: 0.85rem; color: #94a3b8; font-style: italic; }

        /* Modal Styles */
        .modal-overlay { display: none; position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.7); z-index: 50; }
        .modal { display: none; position: fixed; top: 50%; left: 50%; transform: translate(-50%, -50%); background: var(--bg-panel); padding: 25px; border: 1px solid var(--border-color); border-radius: 12px; z-index: 100; width: 90%; max-width: 550px; box-shadow: 0 10px 30px rgba(0,0,0,0.8); }
        .modal h2 { margin-top: 0; color: var(--text-cyan); border-bottom: 1px solid var(--border-color); padding-bottom: 10px; }
        .scroll-area { max-height: 65vh; overflow-y: auto; padding-right: 15px; }
        
        .scroll-area::-webkit-scrollbar { width: 8px; }
        .scroll-area::-webkit-scrollbar-track { background: var(--bg-inner); border-radius: 10px; }
        .scroll-area::-webkit-scrollbar-thumb { background: #4a5568; border-radius: 10px; }
        
        .cfg-section { margin-top: 20px; margin-bottom: 15px; color: #cbd5e1; font-weight: bold; font-size: 1.1rem; }
        .form-grid { display: grid; grid-template-columns: 1.2fr 1fr; gap: 12px; }
        .form-group { margin-bottom: 10px; }
        .form-group.full { grid-column: span 2; }
        .form-group label { display: block; margin-bottom: 5px; font-size: 0.85rem; color: #94a3b8; }
        .form-group input { width: 100%; padding: 8px; border-radius: 6px; border: 1px solid var(--border-color); background: var(--bg-inner); color: white; box-sizing: border-box; }
        
        .modal-footer { display: flex; justify-content: space-between; align-items: center; margin-top: 20px; border-top: 1px solid var(--border-color); padding-top: 15px; }
        .modal-footer-right { display: flex; gap: 10px; }
        .btn-save { background: var(--color-ready); color: black; border: none; padding: 10px 20px; border-radius: 6px; cursor: pointer; font-weight: bold;}
        
        @media (max-width: 768px) { 
            .grid-layout { grid-template-columns: 1fr; } 
            .header { flex-direction: column; gap: 15px; }
            .form-grid { grid-template-columns: 1fr; }
            .form-group.full { grid-column: span 1; }
            .modal-footer { flex-direction: column-reverse; gap: 15px; }
            .modal-footer-right { width: 100%; justify-content: space-between; }
            .btn-restart { width: 100%; }
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>📻 Gateway Radio</h1>
            <div class="btn-group">
                <button class="btn-action" onclick="openConfig()">⚙️ Settings</button>
            </div>
        </div>
        <div class="grid-layout">
            <div class="sidebar">
                <h3 style="color:var(--text-cyan); margin-top:0;">👥 Users (<span id="user-count">0</span>)</h3>
                <ul class="user-list" id="user-list"></ul>
            </div>
            <div class="main-panel">
                <div class="top-status">
                    <div>
                        <div style="font-size:0.8rem; color:#94a3b8;">Current Room:</div>
                        <div class="room-controls">
                            <button class="btn-room" onclick="sendRoomCmd('DOWN')">◀</button>
                            <div style="font-size:1.4rem; color:var(--text-cyan); font-weight:bold;" id="room-name">-</div>
                            <button class="btn-room" onclick="sendRoomCmd('UP')">▶</button>
                        </div>
                    </div>
                </div>
                <div class="display-area">
                    <div id="status-badge" class="status-badge bg-ready">STANDBY</div>
                </div>
            </div>
        </div>
        <div class="footer">Dx Solution HS3PIK</div>
    </div>

    <!-- Configuration Modal -->
    <div class="modal-overlay" id="modal-overlay"></div>
    <div class="modal" id="config-modal">
        <h2>⚙️ Configuration</h2>
        <div class="scroll-area">
            
            <div class="cfg-section">🌐 Server Setup</div>
            <div class="form-grid">
                <div class="form-group full"><label>Server IP / Domain</label><input type="text" id="cfg-ip"></div>
                <div class="form-group"><label>Port</label><input type="number" id="cfg-port"></div>
                <div class="form-group"><label>Server Password</label><input type="text" id="cfg-pass"></div>
                <div class="form-group full"><label>Bot Username</label><input type="text" id="cfg-user"></div>
            </div>

            <div class="cfg-section">📻 Rooms & Passwords</div>
            <div class="form-grid" style="grid-template-columns: 1fr 1fr;">
                <div class="form-group"><label>Room 1 Name</label><input type="text" id="cfg-rm1"></div>
                <div class="form-group"><label>Room 1 Pass</label><input type="text" id="cfg-rm1-p" placeholder="Leave blank if none"></div>
                
                <div class="form-group"><label>Room 2 Name</label><input type="text" id="cfg-rm2"></div>
                <div class="form-group"><label>Room 2 Pass</label><input type="text" id="cfg-rm2-p"></div>
                
                <div class="form-group"><label>Room 3 Name</label><input type="text" id="cfg-rm3"></div>
                <div class="form-group"><label>Room 3 Pass</label><input type="text" id="cfg-rm3-p"></div>
                
                <div class="form-group"><label>Room 4 Name</label><input type="text" id="cfg-rm4"></div>
                <div class="form-group"><label>Room 4 Pass</label><input type="text" id="cfg-rm4-p"></div>
                
                <div class="form-group"><label>Room 5 Name</label><input type="text" id="cfg-rm5"></div>
                <div class="form-group"><label>Room 5 Pass</label><input type="text" id="cfg-rm5-p"></div>
                
                <div class="form-group"><label>Room 6 Name</label><input type="text" id="cfg-rm6"></div>
                <div class="form-group"><label>Room 6 Pass</label><input type="text" id="cfg-rm6-p"></div>
            </div>

            <div class="cfg-section">🎧 Radio Tuning (Timings)</div>
            <div class="form-grid">
                <div class="form-group full">
                    <label>VOX Threshold (ความไวรับเสียง: ค่ามาตรฐาน 400)</label>
                    <input type="number" id="cfg-vox-th" step="50">
                </div>
                <div class="form-group">
                    <label>RX Hang Time (Sec) (หน่วงเวลาฝั่งรับ: แนะนำ 1.0)</label>
                    <input type="number" id="cfg-vox-hang" step="0.1">
                </div>
                <div class="form-group">
                    <label>TX Hang Time (Sec) (หน่วงเวลาฝั่งส่ง: แนะนำ 1.0)</label>
                    <input type="number" id="cfg-tx-hang" step="0.1">
                </div>
            </div>

        </div>
        <div class="modal-footer">
            <button class="btn-action btn-restart" onclick="restartGateway()">🔄 Restart System</button>
            <div class="modal-footer-right">
                <button class="btn-action" onclick="closeConfig()">Cancel</button>
                <button class="btn-save" onclick="saveConfig()">Save & Restart</button>
            </div>
        </div>
    </div>

    <script>
        async function fetchStatus() {
            try {
                const res = await fetch('/api/status'); const data = await res.json();
                const badge = document.getElementById('status-badge');
                badge.innerText = data.state_text; badge.className = 'status-badge ' + data.state_color;
                document.getElementById('room-name').innerText = data.room;
                document.getElementById('user-count').innerText = data.users.length;
                const ul = document.getElementById('user-list'); ul.innerHTML = '';
                data.users.forEach(u => {
                    const li = document.createElement('li'); li.className = 'user-item';
                    li.innerHTML = `<span>🌐 ${u.name}</span> <span class="mic-icon ${u.is_speaking ? 'mic-active' : ''}"></span>`;
                    ul.appendChild(li);
                });
            } catch (e) {}
        }
        
        async function sendRoomCmd(dir) {
            await fetch('/api/change_room', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify({ direction: dir }) });
            fetchStatus(); 
        }

        async function restartGateway() {
            if(confirm("🚨 ยืนยันการรีสตาร์ท Gateway?")) {
                closeConfig();
                document.getElementById('status-badge').innerText = "RESTARTING...";
                document.getElementById('status-badge').className = "status-badge bg-tx";
                fetch('/api/restart', { method: 'POST' });
                setTimeout(() => location.reload(), 10000); 
            }
        }

        // --- Config Functions ---
        async function openConfig() {
            try {
                const res = await fetch('/api/get_config');
                const cfg = await res.json();
                
                document.getElementById('cfg-ip').value = cfg.SERVER_IP || "";
                document.getElementById('cfg-port').value = cfg.PORT || 64738;
                document.getElementById('cfg-pass').value = cfg.PASSWORD || "";
                document.getElementById('cfg-user').value = cfg.USERNAME || "";
                
                document.getElementById('cfg-rm1').value = cfg.ROOM_1 || "";
                document.getElementById('cfg-rm1-p').value = cfg.ROOM_1_PASS || "";
                document.getElementById('cfg-rm2').value = cfg.ROOM_2 || "";
                document.getElementById('cfg-rm2-p').value = cfg.ROOM_2_PASS || "";
                document.getElementById('cfg-rm3').value = cfg.ROOM_3 || "";
                document.getElementById('cfg-rm3-p').value = cfg.ROOM_3_PASS || "";
                document.getElementById('cfg-rm4').value = cfg.ROOM_4 || "";
                document.getElementById('cfg-rm4-p').value = cfg.ROOM_4_PASS || "";
                document.getElementById('cfg-rm5').value = cfg.ROOM_5 || "";
                document.getElementById('cfg-rm5-p').value = cfg.ROOM_5_PASS || "";
                document.getElementById('cfg-rm6').value = cfg.ROOM_6 || "";
                document.getElementById('cfg-rm6-p').value = cfg.ROOM_6_PASS || "";
                
                document.getElementById('cfg-vox-th').value = cfg.VOX_THRESHOLD || 400;
                document.getElementById('cfg-vox-hang').value = cfg.VOX_HANG_TIME || 0.8;
                document.getElementById('cfg-tx-hang').value = cfg.TX_HANG_TIME || 0.7;
                
                document.getElementById('modal-overlay').style.display = 'block';
                document.getElementById('config-modal').style.display = 'block';
            } catch(e) { alert("Cannot load config"); }
        }

        function closeConfig() {
            document.getElementById('modal-overlay').style.display = 'none';
            document.getElementById('config-modal').style.display = 'none';
        }

        async function saveConfig() {
            const newCfg = {
                SERVER_IP: document.getElementById('cfg-ip').value,
                PORT: parseInt(document.getElementById('cfg-port').value),
                PASSWORD: document.getElementById('cfg-pass').value,
                USERNAME: document.getElementById('cfg-user').value,
                ROOM_1: document.getElementById('cfg-rm1').value,
                ROOM_1_PASS: document.getElementById('cfg-rm1-p').value,
                ROOM_2: document.getElementById('cfg-rm2').value,
                ROOM_2_PASS: document.getElementById('cfg-rm2-p').value,
                ROOM_3: document.getElementById('cfg-rm3').value,
                ROOM_3_PASS: document.getElementById('cfg-rm3-p').value,
                ROOM_4: document.getElementById('cfg-rm4').value,
                ROOM_4_PASS: document.getElementById('cfg-rm4-p').value,
                ROOM_5: document.getElementById('cfg-rm5').value,
                ROOM_5_PASS: document.getElementById('cfg-rm5-p').value,
                ROOM_6: document.getElementById('cfg-rm6').value,
                ROOM_6_PASS: document.getElementById('cfg-rm6-p').value,
                VOX_THRESHOLD: parseFloat(document.getElementById('cfg-vox-th').value),
                VOX_HANG_TIME: parseFloat(document.getElementById('cfg-vox-hang').value),
                TX_HANG_TIME: parseFloat(document.getElementById('cfg-tx-hang').value)
            };
            
            try {
                await fetch('/api/save_config', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify(newCfg)
                });
                closeConfig();
                document.getElementById('status-badge').innerText = "APPLYING...";
                document.getElementById('status-badge').className = "status-badge bg-tx";
                setTimeout(() => location.reload(), 12000);
            } catch(e) { alert("Failed to save configuration."); }
        }

        setInterval(fetchStatus, 500); fetchStatus();
    </script>
</body>
</html>
"""

@app.route('/')
def index():
    return render_template_string(HTML_TEMPLATE)

@app.route('/api/change_room', methods=['POST'])
def api_change_room():
    direction = request.json.get('direction')
    if direction in ["UP", "DOWN"]:
        change_room(direction, source="Web")
    return jsonify({"status": "success", "room": ROOM})

@app.route('/api/restart', methods=['POST'])
def api_restart():
    subprocess.Popen(["sudo", "systemctl", "restart", "mumble-gateway.service"])
    return jsonify({"status": "restarting"})

@app.route('/api/get_config', methods=['GET'])
def api_get_config():
    return jsonify(load_config())

@app.route('/api/save_config', methods=['POST'])
def api_save_config():
    new_cfg = request.json
    try:
        with open(CONFIG_FILE, 'w') as f:
            json.dump(new_cfg, f, indent=4)
        subprocess.Popen(["sudo", "systemctl", "restart", "mumble-gateway.service"])
        return jsonify({"status": "success"})
    except Exception as e:
        return jsonify({"status": "error", "message": str(e)}), 500

@app.route('/api/status')
def api_status():
    global mumble, mumble_connected
    if not mumble_connected:
        return jsonify({ "state_text": "SETUP REQUIRED", "state_color": "bg-err", "room": "-", "users": [] })

    state_text, state_color = "STANDBY", "bg-ready"
    if is_transmitting:
        state_text, state_color = "TX ACTIVE", "bg-tx"
    elif is_vox_active:
        state_text, state_color = "RX ACTIVE", "bg-rx"

    users = []
    if 'mumble' in globals() and mumble and mumble.is_alive():
        my_ch_id = next((u['channel_id'] for u in mumble.users.values() if u['name'] == USERNAME), None)
        if my_ch_id is not None:
            for u in mumble.users.values():
                if u['channel_id'] == my_ch_id:
                    name = u['name']
                    is_speaking = (time.time() - user_speaking_status.get(name, 0)) < 0.5
                    users.append({"name": name, "is_speaking": is_speaking})
    
    return jsonify({ "state_text": state_text, "state_color": state_color, "room": ROOM, "users": sorted(users, key=lambda x: x['name']) })

def run_web_server():
    app.run(host='0.0.0.0', port=8080, debug=False, use_reloader=False)

# ==================== MAIN PROGRAM ====================
mumble = None
mumble_connected = False

try:
    check_audio_device()
    wait_for_network()
    _, current_ip = get_network_info()

    threading.Thread(target=run_web_server, daemon=True).start()
    print(f"🌐 Web Monitor running at http://{current_ip}:8080")

    if SERVER_IP:
        try:
            update_display("Connecting...", SERVER_IP)
            
            # ส่งกุญแจ (Tokens) ทั้งหมดเข้าไปตอนเริ่มล็อกอิน
            mumble = Mumble(SERVER_IP, USERNAME, password=PASSWORD, port=PORT, tokens=ROOM_PASSWORDS)
            mumble.start()
            mumble.is_ready()
            mumble.set_receive_sound(True)
            
            target_channel = mumble.channels.find_by_name(ROOM)
            if target_channel: 
                target_channel.move_in()
                
            mumble.callbacks.set_callback("sound_received", sound_received_handler)
            threading.Thread(target=vox_monitor_thread, args=(mumble,), daemon=True).start()
            mumble_connected = True
            update_display("Server Status", "Connected! OK")
        except Exception as e:
            print(f"⚠️ Mumble Connection Error: {e}")
            mumble_connected = False
    else:
        print("⚠️ No Server IP configured.")
        mumble_connected = False

    while True: 
        current_time = time.time()
        
        if not mumble_connected or (mumble and not mumble.is_alive()):
            if is_transmitting:
                GPIO.output(GPIO_PTT, GPIO.LOW)
                is_transmitting = False
            
            dots = "." * (int(current_time * 2) % 4)
            update_display("Setup Required", f"IP: {current_ip}{dots}")
            time.sleep(2)
            continue
        
        if current_time - last_audio_time < TX_HANG_TIME:
            if not is_transmitting:
                GPIO.output(GPIO_PTT, GPIO.HIGH)
                update_display(f"Room: {ROOM}", "TX ACTIVE")
                is_transmitting = True
                last_idle_text = "TX"
        elif is_vox_active:
            if last_idle_text != "RX":
                update_display(f"Room: {ROOM}", "RX ACTIVE")
                last_idle_text = "RX"
        else:
            if is_transmitting:
                GPIO.output(GPIO_PTT, GPIO.LOW)
                is_transmitting = False
            
            dots = "." * (int(current_time * 2) % 4)
            idle_text = f"Ready{dots:<3}" 
            if idle_text != last_idle_text:
                update_display(f"Room: {ROOM}", idle_text)
                last_idle_text = idle_text
                
        time.sleep(0.05) 

except KeyboardInterrupt:
    print("\nStopping program...")
finally:
    if mumble and mumble.is_alive(): mumble.stop()
    GPIO.output(GPIO_PTT, GPIO.LOW)
    GPIO.cleanup()
    if stream_in: 
        try: stream_in.stop_stream(); stream_in.close()
        except: pass
    if stream_out: 
        try: stream_out.stop_stream(); stream_out.close()
        except: pass
    if p: 
        try: p.terminate()
        except: pass
    if oled_enabled and device:
        with lcd_lock: 
            try: device.clear()
            except: pass
    print("System Cleaned and Exited.")


```

**วิธีเข้าหน้า WebUI:** ให้เปิดเบราว์เซอร์ในมือถือหรือคอมพิวเตอร์ที่ต่อเน็ต/Wi-Fi วงเดียวกัน แล้วพิมพ์ URL ว่า `http://[IP-ของ-Pi]:8080` (ดูเลข IP ได้ที่หน้าจอ LCD ของตัวเครื่องเลยครับ)

## บทที่ 6: การตั้งค่าให้ระบบทำงานอัตโนมัติ (Auto-Start Service)

**1. สร้างไฟล์ Service:**
```bash
sudo nano /etc/systemd/system/mumble-gateway.service
```

**2. ใส่โค้ดตั้งค่า (🔴 อย่าลืมเปลี่ยน `[USERNAME]` เป็นชื่อ User ของคุณ ทั้งสองจุดนะครับ):**
```ini
[Unit]
Description=Mumble Radio Gateway (LCD Version)
After=network.target sound.target

[Service]
ExecStart=/usr/bin/python3 /home/[USERNAME]/gateway.py
WorkingDirectory=/home/[USERNAME]/
StandardOutput=inherit
StandardError=inherit
Restart=always
RestartSec=10
User=root

[Install]
WantedBy=multi-user.target
```

**3. เปิดใช้งาน Service:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable mumble-gateway.service
sudo systemctl start mumble-gateway.service
```

## บทที่ 7: 🛡️ ระบบยามสายตรวจป้องกันคีย์ค้าง (TOT Watchdog)

**ปัญหา:** คลื่นวิทยุ (RF) มักจะกวนเข้าบอร์ด Pi ทำให้โปรแกรมหลักค้าง และส่งผลให้วิทยุกดส่งคีย์ค้างยาวตลอดเวลา 
**ทางแก้:** สร้างโปรแกรมแยกอีกตัวเพื่ออ่านสถานะ GPIO โดยตรง หากค้างเกิน 120 วินาที ให้บังคับรีสตาร์ทระบบหลักทันที

**1. สร้างไฟล์โค้ด Watchdog:**
```bash
nano ~/tot_watchdog.py
```

ใส่โค้ดนี้ลงไป:
```python
import time
import subprocess
import sys

# === ตั้งค่า ===
GPIO_PTT = 26
MAX_TX_TIME = 120  # เวลาสูงสุดที่ยอมให้คีย์ค้าง (วินาที)
# ============

def read_gpio_state(pin):
    try:
        with open(f"/sys/class/gpio/gpio{pin}/value", "r") as f:
            return f.read().strip() == "1"
    except:
        return False

print(f"👮‍♂️ TOT Watchdog Started! Monitoring GPIO {GPIO_PTT} for {MAX_TX_TIME} seconds.")
tx_start_time = 0
is_transmitting = False

try:
    while True:
        pin_is_high = read_gpio_state(GPIO_PTT)

        if pin_is_high:
            if not is_transmitting:
                is_transmitting = True
                tx_start_time = time.time()
            else:
                elapsed_time = time.time() - tx_start_time
                if elapsed_time >= MAX_TX_TIME:
                    print(f"🚨 [WARNING] TX Timeout! เตะปลั๊ก Restart Mumble Gateway...")
                    subprocess.run(["sudo", "systemctl", "restart", "mumble-gateway.service"])
                    time.sleep(10)
                    is_transmitting = False
                    tx_start_time = 0
        else:
            if is_transmitting:
                is_transmitting = False
                tx_start_time = 0
                
        time.sleep(1)
except KeyboardInterrupt:
    sys.exit(0)
```

**2. สร้างไฟล์ Service ให้ Watchdog:**
```bash
sudo nano /etc/systemd/system/tot-watchdog.service
```

ใส่โค้ดตั้งค่าดังนี้ (🔴 อย่าลืมเปลี่ยน `[USERNAME]` เป็นชื่อ User ของคุณ):
```ini
[Unit]
Description=TOT Watchdog for Mumble Gateway
After=network.target mumble-gateway.service

[Service]
ExecStart=/usr/bin/python3 /home/[USERNAME]/tot_watchdog.py
WorkingDirectory=/home/[USERNAME]/
StandardOutput=inherit
StandardError=inherit
Restart=always
RestartSec=5
User=root

[Install]
WantedBy=multi-user.target
```

**3. เปิดใช้งาน Watchdog:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable tot-watchdog.service
sudo systemctl start tot-watchdog.service
```

```bash
sudo reboot
```

## บทที่ 8: เทคนิคขั้นสูงและการแก้ปัญหาหน้างาน (Troubleshooting)

หากต้องนำไปใช้งานจริงอย่างหนักหน่วง แนะนำให้เสริมความแข็งแกร่งระดับฮาร์ดแวร์เพื่อป้องกันคลื่นแม่เหล็กไฟฟ้า (RF Interference) ดังนี้:

* **เพิ่มตัวต้านทาน Pull-Down:** นำตัวต้านทาน 10k Ohm ต่อคร่อมระหว่างขา GPIO 26 กับขา GND เพื่อดึงไฟลงเสมอ ป้องกันไม่ให้คีย์ค้างตอนที่ระบบล่ม
* **ติดตั้งแกนเฟอร์ไรต์ (Ferrite Bead):** นำมาหนีบที่สายนำสัญญาณ PTT, สาย Audio และสาย USB เพื่อป้องกัน RF วิ่งย้อนเข้ามากวนบอร์ด Pi
* **ย้ายสายอากาศให้ห่าง:** สายอากาศของวิทยุสื่อสารไม่ควรตั้งอยู่ใกล้บอร์ด Pi ควรลากสายนำสัญญาณ (เช่น RG58) นำเสาออกไปตั้งนอกอาคาร หรือให้อยู่ห่างอย่างน้อย 1-2 เมตรเสมอ

## บทที่ 9: คำสั่งพื้นฐานสำหรับการดูแลระบบ (System Commands)

หลังจากที่คุณตั้งค่าทุกอย่างเสร็จแล้ว ระบบจะทำงานเป็น Background Service หากคุณต้องการเข้าไปตรวจสอบ หรือสั่งการระบบ สามารถใช้คำสั่งเหล่านี้ใน Terminal ได้เลยครับ:

**1. คำสั่งควบคุมระบบ Gateway หลัก (`mumble-gateway.service`)**
* ดูสถานะการทำงาน: `sudo systemctl status mumble-gateway.service`
* สั่งหยุดทำงาน: `sudo systemctl stop mumble-gateway.service`
* สั่งเริ่มทำงานใหม่: `sudo systemctl start mumble-gateway.service`
* สั่งรีสตาร์ทระบบ: `sudo systemctl restart mumble-gateway.service`
* 🖥️ ดู Log การทำงานแบบสดๆ (Real-time): `sudo journalctl -u mumble-gateway.service -f`

**2. คำสั่งควบคุมระบบ Watchdog (`tot-watchdog.service`)**
* ดูสถานะการทำงาน: `sudo systemctl status tot-watchdog.service`
* สั่งหยุดทำงาน: `sudo systemctl stop tot-watchdog.service`
* สั่งเริ่มทำงานใหม่: `sudo systemctl start tot-watchdog.service`
* สั่งรีสตาร์ทระบบ: `sudo systemctl restart tot-watchdog.service`
* 🖥️ ดู Log ว่ามีการเตะปลั๊กตัดคีย์หรือไม่: `sudo journalctl -u tot-watchdog.service -f`

*(หากต้องการออกจากหน้าต่างดู Log แบบสด ให้กดปุ่ม `Ctrl + C` ที่คีย์บอร์ด)*
                         
