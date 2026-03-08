# Week 2 — Platform Deployment with systemd
Deploy a three-tier e-commerce application as Linux systemd 
services on AWS EC2.

## Architecture
```
┌─────────────────┐
│  React Frontend │  (Port 3000)
│   (Tier 1)      │
└────────┬────────┘
         │ HTTP/REST
         │
┌────────▼────────┐
│ Node.js Backend │  (Port 5000)
│   (Tier 2)      │
└────────┬────────┘
         │ SQL
         │
┌────────▼────────┐
│   PostgreSQL    │  (Port 5432)
│   (Tier 3)      │
└─────────────────┘
```


## Tech Stack
- Frontend: React 18
- Backend: Node.js, Express.js
- Database: PostgreSQL 14
- Server: AWS EC2 (Ubuntu 22.04, t2.micro)
- Service Manager: systemd

## What I Did

### 1. Launched EC2 Instance
- Ubuntu 22.04 LTS, t2.micro
- Security group: ports 22, 80, 3000, 5000, 5432
- Storage: 20GB

### 2. Installed Dependencies
```bash
# Node.js via nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
nvm install 20

# PostgreSQL
sudo apt install postgresql postgresql-contrib -y
```

### 3. Configured the Database
```bash
sudo -u postgres psql
CREATE DATABASE ecommercedb;
```

### 4. Configured Backend
Created `backend/.env`:
```
PORT=5000
DB_HOST=localhost
DB_USER=abdul
DB_PASS=your_postgres_password
DB_NAME=ecommercedb
```

### 5. Built Frontend
Fixed hardcoded localhost URL in App.js:
```javascript
// Before (developer hardcoded directly in App.js)
const API = "http://localhost:5000";

// After (correct)
const API = process.env.REACT_APP_API_URL || "http://localhost:5000";
```

```bash
npm run build
```

### 6. Created systemd Service Files

**backend.service** — starts after PostgreSQL, auto-restarts on failure

**frontend.service** — starts after backend, serves React build on port 3000

### 7. Enabled Services on Boot
```bash
sudo systemctl daemon-reload
sudo systemctl enable backend.service
sudo systemctl enable frontend.service
sudo systemctl start backend.service
sudo systemctl start frontend.service
```

### 8. Verified Auto-Start After Reboot
```bash
sudo reboot
# After reboot all services came back automatically
sudo systemctl status postgresql@14-main
sudo systemctl status backend.service
sudo systemctl status frontend.service
```

## Useful systemd Commands Learned
```bash
sudo systemctl start|stop|restart|status <service>
sudo journalctl -u <service> -f        # live logs
sudo journalctl -u <service> -n 50     # last 50 lines
sudo systemctl enable <service>        # auto-start on boot
sudo systemctl daemon-reload           # reload after editing service files
```

## Errors I Encountered & Fixed
| Error | Cause | Fix |
|---|---|---|
| Database "ecommerceDB" does not exist | PostgreSQL lowercased the name | Used lowercase `ecommercedb` in .env |
| frontend.service exit-code 127 | systemd couldn't find serve binary | Added Environment=PATH to service file |
| Products not showing (ERR_CONNECTION_REFUSED) | App.js had localhost hardcoded | Updated App.js to use REACT_APP_API_URL |

## Key Concepts Learned
- systemd manages services — start, stop, restart, boot behavior
- Service files live in `/etc/systemd/system/`
- `After=` and `Requires=` control service dependency order
- `journalctl` is the systemd log viewer for debugging
- Hardcoded values override environment variables and break environment-specific deployments.


## Files
- `Intermediate2/` — application source code
- `backend.service` — systemd service file for Node.js backend
- `frontend.service` — systemd service file for React frontend
