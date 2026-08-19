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
sudo pip3 install pymumble smbus2 pyaudio numpy flask rpi_lcd rpi-lgpio --break-system-packages
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
from rpi_lcd import LCD
from pymumble_py3 import Mumble

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

# ==================== 🔴 CONFIGURATION ====================
SERVER_IP = "192.168.10.20"                 # 🔴 [ต้องใส่] IP หรือ Domain ของ Mumble Server
PORT = 64738                                # 🔴 [ต้องใส่] พอร์ตของเซิร์ฟเวอร์
PASSWORD = "your_password"                  # 🔴 [ต้องใส่] รหัสผ่านเข้าเซิร์ฟเวอร์
USERNAME = "Mumble-Gateway"                 # 🔴 [ต้องใส่] ชื่อที่จะแสดงในแอป Mumble

LCD_ADDRESS = 0x27                          # 🔴 Address ของจอ 16x2 (ส่วนใหญ่ 0x27 หรือ 0x3F)

# --- 🔴 รายชื่อห้องที่จะให้ระบบทำงาน ---
ROOMS = ["CH1", "CH2", "CH3", "CH4"]  # 🔴 [ต้องแก้] เปลี่ยนชื่อห้อง
current_room_idx = 0  
ROOM = ROOMS[current_room_idx]

# --- ตั้งค่า Hardware ---
GPIO_PTT = 26  
GPIO_BTN_UP = 17    
GPIO_BTN_DOWN = 27  

VOX_THRESHOLD = 400    
VOX_HANG_TIME = 0.8    
TX_HANG_TIME = 0.7     

FORMAT = pyaudio.paInt16
CHANNELS = 1
RATE = 48000
CHUNK = 960  
# =======================================================

# --- Safe LCD 16x2 Initialization ---
lcd = None
lcd_enabled = False
lcd_lock = threading.Lock()

try:
    lcd = LCD(address=LCD_ADDRESS, bus=1)
    lcd_enabled = True
    print(f"✅ LCD 16x2 Connected at Address 0x{LCD_ADDRESS:02X}")
except Exception as e:
    print(f"⚠️ Warning: Could not initialize LCD at 0x{LCD_ADDRESS:02X} ({e})")
    print("⚠️ Program will continue running WITHOUT LCD display.")

# ตั้งค่า GPIO
GPIO.setwarnings(False)
GPIO.setmode(GPIO.BCM)
GPIO.setup(GPIO_PTT, GPIO.OUT, initial=GPIO.LOW)
GPIO.setup(GPIO_BTN_UP, GPIO.IN, pull_up_down=GPIO.PUD_UP)
GPIO.setup(GPIO_BTN_DOWN, GPIO.IN, pull_up_down=GPIO.PUD_UP)

def update_display(line1, line2):
    if not lcd_enabled or lcd is None:
        return
    with lcd_lock:
        try:
            # ljust(16)[:16] เติมช่องว่างให้เต็มบรรทัดเพื่อลบตัวอักษรเก่าที่อาจค้างอยู่
            lcd.text(str(line1).ljust(16)[:16], 1)
            lcd.text(str(line2).ljust(16)[:16], 2)
        except Exception as e:
            print(f"⚠️ LCD Write Warning: {e}")

# ==================== BUTTON CALLBACKS ====================
def change_room(direction, source="Button"):
    global current_room_idx, ROOM, mumble
    
    if direction == "UP":
        current_room_idx = (current_room_idx + 1) % len(ROOMS)
    elif direction == "DOWN":
        current_room_idx = (current_room_idx - 1) % len(ROOMS)
        
    ROOM = ROOMS[current_room_idx]
    
    if 'mumble' in globals() and mumble.is_alive():
        target_channel = mumble.channels.find_by_name(ROOM)
        if target_channel:
            target_channel.move_in()
            msg = "Switched!" if source == "Button" else "Switched(Web)!"
            update_display(f"Room: {ROOM}", msg)
            print(f"🔄 Switched to room: {ROOM} (By {source})")

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
    update_display("System Check", "1. Audio (USB)")
    time.sleep(2) 
    
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
            time.sleep(2) 
            break
        else:
            p.terminate() 
            update_display("Error: Audio", "No USB Card")
            time.sleep(2)
            
    stream_out = p.open(format=FORMAT, channels=CHANNELS, rate=RATE, output=True, output_device_index=usb_idx)
    stream_in = p.open(format=FORMAT, channels=CHANNELS, rate=RATE, input=True, frames_per_buffer=CHUNK, input_device_index=usb_idx)

def get_network_info():
    try:
        route_output = subprocess.check_output(f"ip route get {SERVER_IP}", shell=True).decode()
        net_type = "WiFi" if "wlan" in route_output else "LAN" if "eth" in route_output else "Net"
        s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        s.connect((SERVER_IP, PORT))
        ip = s.getsockname()[0]
        s.close()
        return net_type, ip
    except Exception:
        return "Unknown", "No IP"

def wait_for_network():
    update_display("System Check", "2. Network")
    time.sleep(2) 
    
    while True:
        try:
            s = socket.create_connection((SERVER_IP, PORT), timeout=3)
            s.close()
            net_type, ip_address = get_network_info()
            update_display(f"Net: {net_type}", f"{ip_address}")
            time.sleep(5) 
            break 
        except OSError:
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
    <title>Mumble Gateway Monitor</title>
    <style>
        :root {
            --bg-dark: #0f151c; --bg-panel: #1a2332; --bg-inner: #111827;
            --text-main: #e2e8f0; --text-cyan: #00e5ff; --border-color: #2d3748;
            --color-ready: #22c55e; --color-tx: #ef4444; --color-rx: #3b82f6;
        }
        body { font-family: 'Segoe UI', sans-serif; background-color: var(--bg-dark); color: var(--text-main); margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }
        .container { background-color: var(--bg-panel); border: 1px solid var(--border-color); border-radius: 12px; padding: 20px 30px; width: 100%; max-width: 1000px; box-shadow: 0 10px 25px rgba(0,0,0,0.5); }
        .header { text-align: center; border-bottom: 1px solid var(--border-color); padding-bottom: 15px; margin-bottom: 25px; position: relative; }
        .header h1 { color: var(--text-cyan); margin: 0; font-size: 1.8rem; text-shadow: 0 0 10px rgba(0, 229, 255, 0.2); }
        
        .btn-restart {
            position: absolute; right: 0; top: 0;
            background-color: transparent; border: 1px solid #ef4444; color: #ef4444;
            padding: 5px 15px; border-radius: 8px; cursor: pointer; transition: 0.3s;
        }
        .btn-restart:hover { background-color: rgba(239, 68, 68, 0.2); }

        .grid-layout { display: grid; grid-template-columns: 300px 1fr; gap: 20px; }
        .sidebar { background-color: var(--bg-inner); border: 1px solid var(--border-color); border-radius: 8px; padding: 15px; display: flex; flex-direction: column; max-height: 400px; }
        .section-title { color: var(--text-cyan); font-size: 1rem; margin-top: 0; margin-bottom: 15px; }
        .user-list { list-style: none; padding: 0; margin: 0; overflow-y: auto; flex-grow: 1; }
        .user-item { background-color: var(--bg-panel); padding: 10px 15px; margin-bottom: 8px; border-radius: 6px; display: flex; justify-content: space-between; border-left: 3px solid transparent; }
        .mic-icon { width: 12px; height: 12px; border-radius: 50%; background-color: #4a5568; }
        .mic-active { background-color: var(--color-ready); box-shadow: 0 0 8px var(--color-ready); }
        .main-panel { display: flex; flex-direction: column; gap: 15px; }
        .top-status { background-color: var(--bg-inner); border: 1px solid var(--border-color); border-radius: 8px; padding: 15px 20px; display: flex; justify-content: space-between; align-items: center; }
        .room-label { font-size: 0.8rem; color: #94a3b8; }
        .room-name { font-size: 1.4rem; color: var(--text-cyan); font-weight: bold; }
        .room-controls { display: flex; align-items: center; gap: 15px; }
        .btn-room { background-color: var(--bg-panel); border: 1px solid var(--border-color); color: var(--text-cyan); font-size: 1.2rem; padding: 5px 15px; border-radius: 8px; cursor: pointer; }
        .display-area { background-color: #0b111a; border: 1px dashed var(--border-color); border-radius: 8px; flex-grow: 1; display: flex; justify-content: center; align-items: center; min-height: 200px; }
        .status-badge { font-size: 1.8rem; font-weight: bold; padding: 15px 40px; border-radius: 50px; letter-spacing: 2px; }
        .bg-ready { color: var(--color-ready); border: 2px solid var(--color-ready); background: rgba(34, 197, 94, 0.1); }
        .bg-tx { color: var(--color-tx); border: 2px solid var(--color-tx); background: rgba(239, 68, 68, 0.1); }
        .bg-rx { color: var(--color-rx); border: 2px solid var(--color-rx); background: rgba(59, 130, 246, 0.1); }
        .bottom-stats { background-color: var(--bg-inner); border: 1px solid var(--border-color); border-radius: 8px; padding: 15px 20px; display: flex; justify-content: space-around; text-align: center; }
        .stat-group { display: flex; flex-direction: column; gap: 5px; }
        .stat-title { font-size: 0.75rem; color: #94a3b8; }
        .stat-value { font-size: 1rem; color: var(--text-cyan); font-weight: bold; }
        .footer { text-align: right; margin-top: 20px; font-size: 0.75rem; color: #64748b; }
        
        @media (max-width: 768px) { 
            .grid-layout { grid-template-columns: 1fr; } 
            .btn-restart { position: static; display: block; width: 100%; margin-top: 15px; }
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>📻 Gateway Radio Controller</h1>
            <button class="btn-restart" onclick="restartGateway()">🔄 Restart System</button>
        </div>
        <div class="grid-layout">
            <div class="sidebar">
                <h3 class="section-title">👥 Online Users (<span id="user-count">0</span>)</h3>
                <ul class="user-list" id="user-list"></ul>
            </div>
            <div class="main-panel">
                <div class="top-status">
                    <div>
                        <div class="room-label">Current Room:</div>
                        <div class="room-controls">
                            <button class="btn-room" onclick="sendRoomCmd('DOWN')">◀</button>
                            <div class="room-name" id="room-name">-</div>
                            <button class="btn-room" onclick="sendRoomCmd('UP')">▶</button>
                        </div>
                    </div>
                </div>
                <div class="display-area">
                    <div id="status-badge" class="status-badge bg-ready">STANDBY</div>
                </div>
                <div class="bottom-stats">
                    <div class="stat-group"><span class="stat-title">🕒 SYSTEM TIME</span><span class="stat-value" id="clock">--:--:--</span></div>
                    <div class="stat-group"><span class="stat-title">⚙️ SYSTEM STATUS</span><span class="stat-value">ONLINE</span></div>
                </div>
            </div>
        </div>
        <div class="footer">Dx Solution HS3PIK</div>
    </div>
    <script>
        function updateClock() { document.getElementById('clock').innerText = new Date().toLocaleString('th-TH'); }
        setInterval(updateClock, 1000); updateClock();

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
            if(confirm("🚨 ยืนยันการรีสตาร์ท Gateway? (ระบบจะตัดการเชื่อมต่อชั่วคราว)")) {
                try {
                    await fetch('/api/restart', { method: 'POST' });
                    document.getElementById('status-badge').innerText = "RESTARTING...";
                    document.getElementById('status-badge').className = "status-badge bg-tx";
                    setTimeout(() => location.reload(), 15000); 
                } catch(e) { console.error(e); }
            }
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

@app.route('/api/status')
def api_status():
    global mumble
    state_text, state_color = "STANDBY", "bg-ready"
    if is_transmitting:
        state_text, state_color = "TX ACTIVE", "bg-tx"
    elif is_vox_active:
        state_text, state_color = "RX ACTIVE", "bg-rx"

    users = []
    if 'mumble' in globals() and mumble.is_alive():
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
try:
    check_audio_device()
    wait_for_network()

    mumble = Mumble(SERVER_IP, USERNAME, password=PASSWORD, port=PORT)
    mumble.start()
    mumble.is_ready()
    mumble.set_receive_sound(True)
    
    target_channel = mumble.channels.find_by_name(ROOM)
    if target_channel: target_channel.move_in()
        
    mumble.callbacks.set_callback("sound_received", sound_received_handler)
    threading.Thread(target=vox_monitor_thread, args=(mumble,), daemon=True).start()
    threading.Thread(target=run_web_server, daemon=True).start()
    
    _, current_ip = get_network_info()
    print(f"🌐 Web Monitor running at http://{current_ip}:8080")

    while True: 
        current_time = time.time()
        
        if 'mumble' in globals() and not mumble.is_alive():
            if is_transmitting:
                GPIO.output(GPIO_PTT, GPIO.LOW)
                is_transmitting = False
            
            retry = 0
            while not mumble.is_alive() and retry < 5:
                retry += 1
                update_display("Network Lost!", f"Retry: {retry}/5")
                time.sleep(10)
            
            if not mumble.is_alive():
                sys.exit(1)
            else:
                update_display("Server Status", "Connected! OK")
                last_idle_text = ""
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
    if 'mumble' in globals() and mumble.is_alive(): mumble.stop()
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
    if lcd_enabled and lcd:
        with lcd_lock: 
            try: lcd.clear()
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
nano /home/[USERNAME]/tot_watchdog.py
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
                         
