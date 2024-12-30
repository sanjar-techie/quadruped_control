# Multi-Camera Remote Streaming Setup Documentation

## Overview
This documentation outlines the setup of a multi-camera streaming system using a Raspberry Pi 4. Three USB cameras are connected and streamed over the internet using MJPG-streamer and Remote.it, optimized for low latency and performance.

---

## Hardware Setup
- **Raspberry Pi 4**
- **Three USB Cameras**
  - Camera device mappings:
    - Camera 1: `/dev/video0`
    - Front Camera: `/dev/video2`
    - Camera 3: `/dev/video4`

---

## Software Components
1. **MJPG-streamer**: Handles camera capture and streaming.
2. **Remote.it**: Provides secure tunneling through firewalls/NAT.
3. **Web Interface**: HTML page to display all streams simultaneously.

---

## Installation Steps

### 1. Install MJPG-streamer
```bash
sudo apt-get update
sudo apt-get install cmake libjpeg8-dev gcc g++

cd ~
git clone https://github.com/jacksonliam/mjpg-streamer.git
cd mjpg-streamer/mjpg-streamer-experimental
make
sudo make install
```

### 2. Install Remote.it
```bash
curl -LkO https://raw.githubusercontent.com/remoteit/installer/master/scripts/auto-install.sh
chmod +x ./auto-install.sh
sudo ./auto-install.sh
```

### 3. Configure Remote.it Services
Run the installer:
```bash
sudo connectd_installer
```
Add three HTTP services:
- Camera 1: Port **8080**
- Camera 2: Port **8081**
- Camera 3: Port **8082**

---

## Streaming Configuration

### Camera Stream Script (`~/start_streams.sh`)
```bash
#!/bin/bash
export LD_LIBRARY_PATH=/usr/local/lib

# Camera 1 - Low quality, optimized for speed
mjpg_streamer -i "input_uvc.so -d /dev/video0 -r 352x288 -f 30 -q 40 -b 1" \
    -o "output_http.so -p 8080 -w /usr/local/share/mjpg-streamer/www" &

# Front Camera - Better quality
mjpg_streamer -i "input_uvc.so -d /dev/video2 -r 640x480 -f 30 -q 75 -b 1" \
    -o "output_http.so -p 8081 -w /usr/local/share/mjpg-streamer/www" &

# Camera 3 - Low quality, optimized for speed
mjpg_streamer -i "input_uvc.so -d /dev/video4 -r 352x288 -f 30 -q 40 -b 1" \
    -o "output_http.so -p 8082 -w /usr/local/share/mjpg-streamer/www" &
```

### Stream Parameters Explained
- `-r`: Resolution (e.g., 352x288, 640x480)
- `-f`: Frame rate (e.g., 30 fps)
- `-q`: JPEG quality (e.g., 40% for side cameras, 75% for front)
- `-b`: Buffer size (1 for minimal latency)
- `-w`: Web server root directory

---

## Web Interface
Create a file named `cameras.html` on your local computer:

```html
<!DOCTYPE html>
<html>
<head>
    <title>All Cameras</title>
    <style>
        body {
            margin: 0;
            padding: 20px;
            background: #1a1a1a;
            color: white;
        }
        .camera-container {
            display: grid;
            grid-template-columns: 1fr 2fr 1fr;
            gap: 20px;
            max-width: 1800px;
            margin: 0 auto;
        }
        .camera {
            background: #2a2a2a;
            padding: 10px;
            border-radius: 8px;
            text-align: center;
        }
        .side-camera iframe {
            width: 100%;
            height: 288px;
            border: none;
        }
        .main-camera iframe {
            width: 100%;
            height: 480px;
            border: none;
        }
    </style>
</head>
<body>
    <div class="camera-container">
        <!-- Replace URLs with your Remote.it URLs -->
        <div class="camera side-camera">
            <h2>Camera 1</h2>
            <iframe src="CAMERA1_URL/javascript_simple.html"></iframe>
        </div>
        <div class="camera main-camera">
            <h2>Front Camera</h2>
            <iframe src="CAMERA2_URL/javascript_simple.html"></iframe>
        </div>
        <div class="camera side-camera">
            <h2>Camera 3</h2>
            <iframe src="CAMERA3_URL/javascript_simple.html"></iframe>
        </div>
    </div>
</body>
</html>
```

---

## Technical Workflow

1. **Camera Capture Process**
   - MJPG-streamer captures frames from USB cameras.
   - Frames are compressed using MJPEG format.
   - Each camera stream runs on a separate port.

2. **Network Flow**
   ```plaintext
   [USB Camera] → [MJPG-streamer] → [Local Port] → [Remote.it Tunnel] → [Remote.it Proxy] → [Web Browser]
   ```

3. **Latency Optimization**
   - Reduced resolution for side cameras (352x288).
   - Lower JPEG quality (40% for side cameras, 75% for front camera).
   - Minimal buffer size (`-b 1`).
   - MJPEG format for efficient streaming.

---

## Usage

1. Start the streams on the Raspberry Pi:
   ```bash
   ./start_streams.sh
   ```
2. Get Remote.it URLs for each camera from the dashboard.
3. Update the `cameras.html` file with your Remote.it URLs.
4. Open `cameras.html` in your web browser to view all streams.

---

## Troubleshooting

1. **Check if streams are running:**
   ```bash
   ps aux | grep mjpg
   ```
2. **Verify ports are listening:**
   ```bash
   netstat -tuln | grep '808'
   ```
3. **View camera capabilities:**
   ```bash
   v4l2-ctl --device=/dev/videoX --list-formats-ext
   ```
4. **Restart streams:**
   ```bash
   pkill mjpg_streamer
   ./start_streams.sh
   ```

---

# Remote Access Setup for Raspberry Pi Using Remote.it

## Overview
This documentation covers the setup of remote access to a Raspberry Pi using Remote.it, allowing SSH access from anywhere, even when the Pi is behind a mobile hotspot or NAT.

---

## Prerequisites
- Raspberry Pi running Ubuntu
- Internet connection (works with mobile hotspot)
- Remote.it account (free)

---

## Two-Part Setup

### Part 1: IP Notification System
This system sends an email whenever the Pi's IP address changes.

#### Create the IP notification script:

```bash
nano /home/beth2/ip_notifier.py
```

#### Script content:

```python
import smtplib
from email.mime.text import MIMEText
import subprocess
import time

def get_ip():
    try:
        ip = subprocess.check_output(['hostname', '-I']).decode('utf-8').strip()
        return ip
    except:
        return None

def send_email(ip_address):
    sender_email = "your_email@gmail.com"
    sender_password = "YOUR_APP_PASSWORD"
    receiver_email = "your_email@gmail.com"
    
    msg = MIMEText(f"Raspberry Pi IP address has changed to: {ip_address}")
    msg['Subject'] = 'Raspberry Pi IP Change Alert'
    msg['From'] = sender_email
    msg['To'] = receiver_email
    
    try:
        server = smtplib.SMTP_SSL('smtp.gmail.com', 465)
        server.login(sender_email, sender_password)
        server.send_message(msg)
        server.quit()
        print(f"Email sent - New IP: {ip_address}")
    except Exception as e:
        print(f"Error sending email: {e}")

def main():
    last_ip = get_ip()
    print(f"Starting IP monitor. Current IP: {last_ip}")
    
    while True:
        current_ip = get_ip()
        if current_ip != last_ip and current_ip is not None:
            send_email(current_ip)
            last_ip = current_ip
        time.sleep(60)

if __name__ == "__main__":
    main()
```

#### Make the script run at startup:

```bash
crontab -e
```

Add the line:
```bash
@reboot python3 /home/beth2/ip_notifier.py &
```

---

### Part 2: Remote.it Setup

#### Install Remote.it:

```bash
curl -LkO https://raw.githubusercontent.com/remoteit/installer/master/scripts/auto-install.sh
chmod +x ./auto-install.sh
sudo ./auto-install.sh
```

#### Configure Remote.it:

```bash
sudo connectd_installer
```

##### Configuration steps:
1. Choose option **1** to sign in with username/password.
2. Enter Remote.it account credentials.
3. Name your device (e.g., `techie`).
4. Choose option **1** to attach a service.
5. Select **SSH** (option **1**).
6. Exit the installer.

---

## How It Works

### IP Notification System
- The script monitors the Pi's IP address every minute.
- When an IP change is detected, it sends an email using Gmail's SMTP server.
- The script automatically starts when the Pi boots up thanks to crontab.

### Remote.it Connection

#### Architecture:
- Remote.it creates a secure tunnel between your Pi and their proxy servers.
- When you connect, you're routed through Remote.it's infrastructure, bypassing NAT and firewall restrictions.

#### Connection Flow:
1. Pi maintains persistent connection to Remote.it servers.
2. When you request access via Remote.it web interface:
   - Remote.it generates a temporary proxy address (e.g., `proxy21.rt3.io:33257`).
   - Your SSH client connects to this proxy.
   - The proxy forwards traffic to your Pi through the established tunnel.

#### Security Features:
- End-to-end encryption
