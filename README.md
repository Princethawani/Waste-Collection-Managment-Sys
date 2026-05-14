# 🗑 Waste Collection Managment Sys — Smart Waste Bin Monitor

A full-stack IoT dashboard for monitoring waste bin fill levels using an Arduino HC-SR04 ultrasonic sensor. Tracks distance in real time, calculates fill percentage, sends email alerts, and auto-schedules bin collections.

---

## Project Structure

```
project/
├── dashboard.html              Single-file frontend (open in any browser)
└── backend/
    ├── src/
    │   ├── server.ts           Express API — routes, serial port, bin logic
    │   ├── auth.ts             JWT authentication + bcrypt password hashing
    │   ├── mailer.ts           Nodemailer email alert engine
    │   └── types.ts            Shared TypeScript interfaces
    ├── test-email.ts           SMTP connection tester
    ├── .env.example            Environment variable template
    ├── package.json
    └── tsconfig.json
```

---

## How It Works

The HC-SR04 sensor sits at the **top** of the bin, facing down. It measures the distance to the waste surface:

- **Large distance** → bin nearly empty → low fill %
- **Small distance** → bin nearly full → high fill %

```
Sensor (top)
    │
    │  distance = 35 cm
    ▼
[waste surface]
════════════════  ← bin bottom
```

Fill % is calculated as:

```
fillPercent = ((binDepth - distance) / binDepth) × 100
```

Where `binDepth` is the internal height of your bin in cm (configurable in the dashboard, default 50 cm).

---

## Quick Start

### 1. Install Dependencies

```bash
cd project/backend
npm install
```

### 2. Configure Environment

```bash
cp .env.example .env
nano .env
```

Your `.env` should look like:

```dotenv
PORT=9001
JWT_SECRET=change-this-to-a-long-random-secret-key
SMTP_HOST=mail.princethawani.com
SMTP_PORT=587
SMTP_USER=info@princethawani.com
SMTP_PASS=your-cpanel-email-password
SERIAL_PORT=/dev/ttyUSB0
```

### 3. Test Email (recommended)

```bash
npx ts-node test-email.ts
```

Expected output:
```
SMTP connection OK - sending test email...
Email sent! <message-id@yourdomain.com>
```

### 4. Start the Backend

```bash
npm run dev
```

Server starts on `http://localhost:9001`

### 5. Open the Dashboard

Open `dashboard.html` directly in your browser — no web server needed.

---

## Default Login

| Username | Password  |
|----------|-----------|
| `admin`  | `admin123` |

> Change this in `auth.ts` → `seedAdmin()` before deploying to production.

---

## Dashboard Features

### Live Bin Status
- Animated bin visual that fills up with colour (green → yellow → red)
- Real-time fill percentage and distance reading
- Status badge: **Safe** / **Close** / **Critical**
- Connection indicator (Live / Offline)

### Sensor Simulator
Use the slider to manually send distance readings without an Arduino connected. Hit **Auto** to simulate continuous sensor data automatically.

### Charts
- **Fill Level Over Time** — line chart of the last 30 readings
- **Status Distribution** — doughnut chart of safe / close / critical counts

### Collection Schedule
- Manually schedule a bin collection with date, time, and notes
- Statuses flow: `pending → confirmed → completed` or `cancelled`
- **Auto-scheduling**: when the bin reaches ≥ 80% full, a collection is automatically scheduled for the next day at 08:00 and an email is sent
- Overdue pending schedules are highlighted in red

### Alert & Sensor Configuration

| Setting | Description |
|---|---|
| Alert Email | Where alert emails are delivered |
| Bin Depth (cm) | Internal height of your bin — used to calculate fill % |
| Critical Threshold | Distance (cm) that triggers a CRITICAL alert |
| Close Threshold | Distance (cm) that triggers a CLOSE warning |
| Enable Alerts | Master toggle for all email alerts |

---

## API Reference

All routes except `/auth/*` and `/health` require:
```
Authorization: Bearer <token>
```

### Auth

| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| POST | `/auth/register` | `{username, email, password}` | Create account |
| POST | `/auth/login` | `{username, password}` | Login, returns JWT |

### Readings

| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| GET | `/readings` | — | Last 100 readings |
| GET | `/readings/latest` | — | Most recent reading |
| POST | `/readings` | `{distance: number}` | Push a new reading |

### Schedules

| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| GET | `/schedules` | — | All schedules |
| POST | `/schedules` | `{scheduledFor, notes}` | Create schedule |
| PATCH | `/schedules/:id` | `{status, notes}` | Update schedule |
| DELETE | `/schedules/:id` | — | Delete schedule |

### Alert Config

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/alerts/config` | Get current config |
| PUT | `/alerts/config` | Update config |

### Health

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/health` | Server health check |

---

## Arduino Integration

### USB Serial (default)

The backend reads the Arduino output automatically over USB. Your sketch prints:
```
Distance: 23
```
The backend strips non-numeric characters and parses the number — this format works as-is with no changes to the Arduino code.

### Serial Port Path

| OS | Typical path |
|----|-------------|
| Linux | `/dev/ttyUSB0` or `/dev/ttyACM0` |
| macOS | `/dev/cu.usbserial-*` |
| Windows | `COM3`, `COM4`, etc. |

Set `SERIAL_PORT` in `.env` to match. If unavailable, the server still starts — use the simulator.

### WiFi (ESP8266 / WiFi Shield)

To send readings over the network instead of USB:

```cpp
HTTPClient http;
http.begin("http://YOUR_SERVER_IP:9001/readings");
http.addHeader("Content-Type", "application/json");
http.addHeader("Authorization", "Bearer YOUR_JWT_TOKEN");
http.POST("{\"distance\": " + String(distance) + "}");
http.end();
```

---

## Email Alerts

Emails go **from** `SMTP_USER` **to** the Alert Email set in the dashboard.

| Trigger | Subject |
|---------|---------|
| Distance below critical threshold | `[ALERT] Bin Sensor - CRITICAL` |
| Distance below close threshold | `[ALERT] Bin Sensor - CLOSE` |
| Bin reaches ≥ 80% full | `[ACTION] Bin Collection Scheduled - XX% full` |

- Distance alerts have a **60-second cooldown**
- Schedule alerts have a **10-minute cooldown**

### cPanel Email Setup

```dotenv
SMTP_HOST=mail.yourdomain.com
SMTP_PORT=587
SMTP_USER=info@yourdomain.com
SMTP_PASS=your-cpanel-email-password
```

Find your exact SMTP host in cPanel → **Email Accounts → Connect Devices → Mail Client Manual Settings**.

---

## npm Scripts

```bash
npm run dev      # Run with ts-node (development)
npm run build    # Compile TypeScript → dist/
npm start        # Run compiled JS (production)
```

---

## Security Notes

- Change `JWT_SECRET` to a long random string before deploying
- Change the default `admin` / `admin123` credentials
- Data is stored in memory and resets on restart — add a database (PostgreSQL, SQLite) for production
- Restrict CORS origins in `server.ts` for production deployments

---

## Troubleshooting

**`Missing credentials for "PLAIN"`**
`dotenv.config()` must be the very first line in `server.ts`, before all other imports. Without it, `SMTP_USER` and `SMTP_PASS` are empty at runtime.

**`ECONNREFUSED` on SMTP**
Wrong `SMTP_HOST`. Check cPanel → Email Accounts → Connect Devices for the exact hostname.

**Serial port not available**
Arduino not plugged in, or wrong port path. Find it with `ls /dev/tty*` on Linux/macOS. The server still starts and the simulator works.

**Dashboard shows "Offline"**
Backend is not running or the API URL is wrong. The dashboard defaults to `http://localhost:9001`. To point it at a different server, open the browser console and run:
```js
localStorage.setItem("apiBase", "http://YOUR_SERVER_IP:9001")
```
Then refresh the page.

---

## Built With

| Component | Technology |
|-----------|-----------|
| Sensor | Arduino HC-SR04 ultrasonic + buzzer |
| Backend language | TypeScript |
| API framework | Express.js |
| Serial communication | SerialPort |
| Email | Nodemailer |
| Password hashing | bcryptjs |
| Authentication | JSON Web Tokens (JWT) |
| Frontend charts | Chart.js |
| Frontend | Vanilla HTML / CSS / JavaScript (single file) |

---

*Built by Enter your Name*