# OCDE Cyber Threat Map

A real-time 3D visualization system that displays firewall threat data on an interactive globe. Designed for Network Operations Center (NOC) wall displays, the system ingests Palo Alto firewall DENY logs via UDP syslog, performs IP geolocation, and renders animated arcs from attack origins to your network location.

![2026-02-12_13-10-08](https://github.com/user-attachments/assets/0e1011cd-6677-4448-a931-510d8ea4ea56)

## Features

- **Real-time Visualization** - Animated arcs show attacks as they happen on a 3D WebGL globe
- **IP Geolocation** - MaxMind GeoLite2 database maps source IPs to geographic coordinates
- **Threat Classification** - Color-coded arcs by threat type (malware, intrusion, DDoS, deny)
- **NOC-Optimized Display** - Dark theme with high-contrast colors, readable from 20+ feet
- **Live Statistics** - Top attacking countries, threat types, and attacks per minute
- **Secure Authentication** - Session-based login with bcrypt password hashing
- **Threat Feed Ticker** - Scrolling advisory ticker with severity colors, configurable via API and admin panel
- **Admin Panel** - Change passwords, customize headings, upload logo, tune display settings
- **Alternative Views** - Toggle between 3D globe and 2D flat map

## Prerequisites

- **Node.js 18.x** or higher
- **MaxMind GeoLite2-City database** (free, requires account)
- **Firewall** configured to send syslog over UDP
- Root privileges for port 514 (or use alternative port)

## Quick Start

### 1. Clone and Install

```bash
Comming Soon!
```

### 2. Download MaxMind Database

1. Create a free account at [MaxMind](https://www.maxmind.com/en/geolite2/signup)
2. Download **GeoLite2-City.mmdb**
3. Place it in the `data/` directory:
   ```bash
   mkdir -p data
   mv ~/Downloads/GeoLite2-City.mmdb data/
   ```

### 3. Configure Environment

```bash
cp .env.example .env
```

Edit `.env` with your settings:

```env
# Required for production
SESSION_SECRET=your-64-character-random-string
DASHBOARD_PASSWORD=YourSecurePassword123!

# Optional
DASHBOARD_USERNAME=admin
SYSLOG_PORT=514
NODE_ENV=development
```

Generate a secure session secret:
```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

### 4. Start the Server

**Development mode** (port 5514, no root required):
```bash
SYSLOG_PORT=5514 npm run dev
```

**Production mode** (port 514, requires root):
```bash
# Option 1: Run with sudo
sudo $(which node) src/app.js

# Option 2: Grant capability (recommended)
sudo setcap cap_net_bind_service=+ep $(which node)
npm start
```

### 5. Access the Dashboard

### !!!WARNING!!!
The Threat Map meant to sit behind a TLS Proxy,  Exposing this to the Internet is highly DISCOURAGED.

1. Open http://localhost:3000
2. Login with your configured credentials
3. The dashboard will display attacks in real-time

## Firewall Configuration

Configure your firewall to send threat logs to the server:

1. **Device > Server Profiles > Syslog**
   - Add server with your receiver's IP address
   - Port: 514 (or your configured port)
   - Transport: UDP
   - Format: **IETF (RFC 5424)** - NOT BSD

2. **Objects > Log Forwarding**
   - Create a profile that forwards THREAT logs to your syslog server

3. **Policies > Security**
   - Apply the log forwarding profile to your security rules

## Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Palo Alto    │ ──▶ │ UDP Receiver │ ──▶ │   Parser     │ ──▶ │  Enrichment  │
│ Firewall     │     │ (Port 514)   │     │ (RFC 5424)   │     │  (MaxMind)   │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
                                                                      │
                                                                      ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  3D Globe    │ ◀── │  Dashboard   │ ◀── │  WebSocket   │ ◀── │  Event Bus   │
│  (Globe.GL)  │     │  (Browser)   │     │  Broadcast   │     │  (enriched)  │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
```

## Project Structure

```
OCDEThreatMap/
├── src/
│   ├── app.js                 # Application entry point
│   ├── receivers/             # UDP syslog listener
│   ├── parsers/               # RFC 5424 / Palo Alto parser
│   ├── enrichment/            # MaxMind geolocation + caching
│   ├── websocket/             # Real-time broadcast to clients
│   ├── middleware/            # Session, auth, rate limiting
│   ├── routes/                # Login, logout, admin APIs
│   └── utils/                 # Security utilities
├── public/
│   ├── dashboard.html         # Main visualization page
│   ├── admin.html             # Admin panel
│   ├── css/                   # NOC-optimized dark theme
│   └── js/                    # Globe.GL, D3, WebSocket client
├── data/
│   └── GeoLite2-City.mmdb     # MaxMind database (download separately)
└── test/                      # Parser tests
```

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `SESSION_SECRET` | Yes (prod) | fallback | 32+ character secret for session signing |
| `DASHBOARD_USERNAME` | No | `admin` | Login username |
| `DASHBOARD_PASSWORD` | Yes (prod) | `ChangeMe` | Login password |
| `SYSLOG_PORT` | No | `514` | UDP port for syslog receiver |
| `SYSLOG_BIND_ADDRESS` | No | `127.0.0.1` | UDP bind address (`0.0.0.0` for remote) |
| `HTTP_PORT` | No | `3000` | HTTP server listen port |
| `HTTP_BIND_ADDRESS` | No | `127.0.0.1` | HTTP bind address (`0.0.0.0` for remote) |
| `THREAT_FEED_API_KEY` | No | — | API key for threat feed / alerts / phishing POST endpoints |
| `OCDE_IP_RANGES` | No | — | Comma-separated CIDR ranges for target detection |
| `TRUST_PROXY` | No | `false` | Set `true` behind a reverse proxy |
| `NODE_ENV` | No | `development` | Set to `production` for secure cookies |

## Security Features

- **bcrypt** password hashing (12 rounds)
- **Rate limiting** on login (5 attempts/15 min) and API routes
- **Helmet.js** security headers
- **httpOnly, sameSite=lax** session cookies
- **Constant-time** password comparison (timing attack prevention)
- **Security event logging** for auditing

## Testing

Send a test syslog message:

```bash
# Start server on alternative port
SYSLOG_PORT=5514 node src/app.js &

# Send test message
echo '<14>1 2024-01-26T10:00:00Z PA-VM - - - [meta src=203.0.113.50 dst=10.0.0.1 action=deny threat_type=malware]' | nc -u localhost 5514
```

Run parser tests:
```bash
node test/test-parser.js
```

## Building Standalone Executables

Create self-contained binaries that run without Node.js installed.

### Build All Platforms

```bash
npm run build
```

This creates executables for Linux, macOS, and Windows in the `dist/` directory.

### Build Single Platform

```bash
npm run build:linux    # Linux x64
npm run build:macos    # macOS x64
npm run build:windows  # Windows x64
```

### Distribution Structure

After building, each platform folder contains:

```
dist/
├── linux/
│   ├── ocde-threat-map-linux    # Executable
│   ├── public/                   # Web assets (required)
│   ├── data/                     # Place GeoLite2-City.mmdb here
│   ├── .env.example              # Configuration template
│   ├── start.sh                  # Launch script
│   └── QUICKSTART.md             # Quick reference
├── macos/
│   └── ...
└── windows/
    ├── ocde-threat-map-win.exe
    ├── start.bat
    └── ...
```

### Deploying a Build

1. Copy the entire platform folder to target machine
2. Download MaxMind GeoLite2-City.mmdb into the `data/` folder
3. Copy `.env.example` to `.env` and configure
4. Run the executable or use the start script

**Linux/macOS:**
```bash
chmod +x ocde-threat-map-linux start.sh
./start.sh
# Or directly: ./ocde-threat-map-linux
```

**Windows:**
```cmd
start.bat
REM Or directly: ocde-threat-map-win.exe
```

### Build Requirements

- Node.js 18.x+ (for building only - not needed to run)
- npm packages installed (`npm install`)
- ~500MB disk space for all platforms

## Admin Panel

Access the admin panel at `/admin` (requires authentication):

- **Change Password** - Update admin password with complexity requirements
- **Customize Heading** - Change the dashboard title
- **Upload Logo** - Replace the default logo with your organization's branding
- **Network Binding** - Configure HTTP and syslog bind addresses and ports
- **Arc Display Limit** - Control the max number of arcs on the globe (1-50, default 20)
- **Ticker Display Limit** - Control the max items in the threat feed ticker (1-20, default 10)

All settings persist to `data/settings.json` and survive server restarts.

## Threat Feed Ticker

A scrolling ticker at the bottom of the dashboard displays threat intelligence advisories. Items follow a FIFO (first in, first out) model — when new items are posted, the oldest items drop off.

### Configuration

**Environment variable** (required to accept POST requests):

```env
THREAT_FEED_API_KEY=your-secret-api-key-here
```

**Admin panel setting:** The ticker display limit (1-20 items) is configurable under Display Settings in the admin panel. Changes apply immediately to all connected dashboards.

### API Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/threat-feed` | None | Returns current ticker items (public) |
| POST | `/api/threat-feed` | API Key | Add new items (via `X-API-Token` header) |
| DELETE | `/api/threat-feed/:id` | Session | Remove an item (admin login required) |

### POST Request Format

Send a single item:

```bash
curl -X POST http://localhost:3000/api/threat-feed \
  -H "Content-Type: application/json" \
  -H "X-API-Token: your-secret-api-key-here" \
  -d '{
    "text": "CVE-2025-1234: Critical RCE in PAN-OS GlobalProtect - Patch immediately",
    "severity": "critical",
    "source": "CISA KEV",
    "expiresAt": "2025-03-01T00:00:00.000Z"
  }'
```

Send multiple items:

```bash
curl -X POST http://localhost:3000/api/threat-feed \
  -H "Content-Type: application/json" \
  -H "X-API-Token: your-secret-api-key-here" \
  -d '[
    {
      "text": "APT29 targeting education sector with spearphishing",
      "severity": "high",
      "source": "CISA Alert"
    },
    {
      "text": "Monthly vulnerability scan scheduled for this weekend",
      "severity": "low",
      "source": "OCDE SOC"
    }
  ]'
```

### Item Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `text` | Yes | string | Advisory text displayed in the ticker (max 500 chars) |
| `severity` | No | string | `critical`, `high`, `medium`, or `low` (default: `medium`) |
| `source` | No | string | Source attribution (max 100 chars, default: `N8N`) |
| `expiresAt` | No | ISO 8601 | Auto-remove after this time; omit or `null` for no expiration |

### Severity Colors

| Severity | Color | Hex |
|----------|-------|-----|
| Critical | Red | `#ff4444` |
| High | Orange | `#ff8c00` |
| Medium | Yellow | `#ffff00` |
| Low | Green | `#00ff00` |

### N8N Integration

The ticker is designed to accept feeds from [N8N](https://n8n.io/) automation workflows. Set up an HTTP Request node in N8N to POST advisories from threat intel sources (CISA KEV, vendor PSIRTs, internal SOC alerts) to `/api/threat-feed` with your API key in the `X-API-Token` header.

Demo items are displayed automatically when no real feed data exists.

## Alerts Panel

A dedicated alerts panel displays environment-level alerts on the dashboard. Uses the same API key as the threat feed.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/alerts` | None | Returns current alerts (public) |
| POST | `/api/alerts` | API Key | Add alerts (via `X-API-Token` header) |
| DELETE | `/api/alerts/:id` | Session | Remove an alert (admin login required) |

```bash
curl -X POST http://localhost:3000/api/alerts \
  -H "Content-Type: application/json" \
  -H "X-API-Token: your-secret-api-key-here" \
  -d '{"text": "Core switch upgrade tonight 10PM-2AM", "severity": "medium", "source": "NOC"}'
```

## Phishing Reports Panel

Tracks phishing email report counts and recent subjects. Integrates with email security automation.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/phishing-reports` | None | Returns total count and recent subjects |
| POST | `/api/phishing-reports` | API Key | Record a new phishing report |

```bash
curl -X POST http://localhost:3000/api/phishing-reports \
  -H "Content-Type: application/json" \
  -H "X-API-Token: your-secret-api-key-here" \
  -d '{"subject": "Urgent: Verify your account credentials"}'
```

## Troubleshooting

**Port 514 permission denied:**
```bash
# Grant Node.js capability to bind privileged ports
sudo setcap cap_net_bind_service=+ep $(which node)
```

**No attacks appearing:**
- Verify firewall is sending to correct IP/port
- Check syslog format is RFC 5424 (IETF), not BSD
- Ensure DENY/threat logs are being forwarded
- Check browser console for WebSocket connection errors

**WebSocket authentication failed:**
- Clear browser cookies and re-login
- Verify SESSION_SECRET matches between restarts

## License

ISC

## Acknowledgments

- [Globe.GL](https://globe.gl/) - 3D globe visualization
- [MaxMind GeoLite2](https://dev.maxmind.com/geoip/geolite2-free-geolocation-data) - IP geolocation
- [Three.js](https://threejs.org/) - WebGL rendering engine
- [D3.js](https://d3js.org/) - 2D flat map geo projections
