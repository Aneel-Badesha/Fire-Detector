# Fire Alarm System - Raspberry Pi Server

## Overview
This server receives fire alarm alerts from ESP32 sensors via HTTP POST requests and displays urgent GUI popup alerts on the Raspberry Pi desktop.

## Features
- Flask HTTP server listening on port 5000
- POST /alert endpoint to receive fire sensor notifications
- GET / status endpoint for health checks
- Automatic GUI popup alerts using zenity when fire is detected
- Logs all alerts to console and server.log

## Setup

### 1. Virtual Environment (Already configured)
```bash
cd /home/abadesha/Documents
. venv/bin/activate
```

### 2. Dependencies Installed
- Flask (Python web framework)
- libnotify-bin (notification tools)
- zenity (GUI dialog tool)

## Running the Server

### Manual Start
```bash
cd /home/abadesha/Documents
. venv/bin/activate
python server.py
```

### Background Start (Current method)
```bash
cd /home/abadesha/Documents
. venv/bin/activate
nohup python server.py > server.log 2>&1 &
```

### Stop the Server
```bash
pkill -f "python.*server.py"
```

### View Logs
```bash
tail -f /home/abadesha/Documents/server.log
```

## API Endpoints

### GET /
Returns server status
```bash
curl http://192.168.1.90:5000/
# Response: {"status":"ok","message":"server running"}
```

### POST /alert
Receives fire alarm notifications from ESP32
```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"msg":"Fire detected!","sensor":"ESP32-001"}' \
  http://192.168.1.90:5000/alert
# Response: {"status":"ok"}
```

## ESP32 Integration

Your ESP32 fire detector sends HTTP POST requests to:
```
URL: http://192.168.1.90:5000/alert
Method: POST
Content-Type: application/json
Body: {"msg":"Alarm Triggered","sensor":"ESP32"}
```

### Actual ESP-IDF Implementation
The fire detector project uses ESP-IDF native HTTP client (not Arduino):

**WiFi Configuration** (`main/fire_detector.c`):
```c
const char* ssid = "BadeshaHome";
const char* password = "Canucks@2011";
wifi_set_rpi_address("192.168.1.90", 5000);
```

**Alert Trigger Logic**:
- Sensor polled every 1 second
- 5-second rolling average calculated from circular buffer
- Alert sent when average flame intensity ≥ 20
- HTTP POST with JSON: `{"msg":"Alarm Triggered"}`
- Mutex-protected state to prevent duplicate alerts

**Task Architecture**:
1. `sensor_read_task`: Polls GPIO34 ADC, updates rolling average
2. `send_message_task`: Sends HTTP POST when alarm triggered
3. `led_flash_task`: Flashes LED indicator during alarm
4. `button_reset_task`: Manual reset via GPIO23 button
```

## Firewall Configuration
Port 5000 is allowed through UFW:
```bash
sudo ufw status
# Output should show: 5000/tcp ALLOW Anywhere
```

## Server Status
- **Running**: Yes (background process)
- **Port**: 5000
- **Accessible from**: All network interfaces (0.0.0.0)
- **Local URL**: http://127.0.0.1:5000
- **Network URL**: http://192.168.1.90:5000

## GUI Alert Details
When a fire alarm is received:
1. Server logs the alert to console and server.log
2. A GUI popup appears on the Raspberry Pi desktop with:
   - Title: "🔥 FIRE ALARM TRIGGERED 🔥"
   - Message showing the fire details
   - Sensor ID that triggered the alarm
   - Warning to check the system immediately

## Troubleshooting

### Check if server is running
```bash
ps aux | grep server.py | grep -v grep
```

### Check if port 5000 is listening
```bash
ss -ltnp | grep :5000
```

### Test from another computer on the network
```bash
curl http://192.168.1.90:5000/
```

### Check active zenity popups
```bash
ps aux | grep zenity | grep -v grep
```

### Close all alert popups
```bash
pkill zenity
```

## Auto-start on Boot (Optional)
To make the server start automatically when the Raspberry Pi boots, you can create a systemd service. Let me know if you need this configured.

## Environment Variables
- `PORT`: Override the listening port (default: 5000)
  ```bash
  PORT=5001 python server.py
  ```

## Notes
- The server uses Flask's development server (fine for local network use)
- For production deployments, consider using gunicorn or uwsgi
- GUI popups require an active desktop session (DISPLAY environment)
- Multiple alerts will create multiple popup windows
