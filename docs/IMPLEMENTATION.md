# AdGuard Home Sync - Production Implementation

## 📋 Table of Contents
- [Environment Overview](#environment-overview)
- [Implementation Details](#implementation-details)
- [Configuration](#configuration)
- [Service Management](#service-management)
- [Monitoring & Troubleshooting](#monitoring--troubleshooting)
- [API Endpoints](#api-endpoints)
- [Maintenance](#maintenance)
- [Security Considerations](#security-considerations)
- [Recovery Procedures](#recovery-procedures)

## 🌐 Environment Overview

### Network Architecture
All servers are Proxmox LXC containers running on the same physical host.

| Server | IP Address | Port | Role | Version |
|--------|------------|------|------|---------|
| **AdGuard Primary** | 10.10.20.6 | 80 | Origin (Master) | v0.107.67 |
| **AdGuard Replica** | 10.10.20.7 | 80 | Replica (Slave) | v0.107.67 |
| **Sync Server** | 10.10.20.8 | 8080 | Sync Controller | Latest |

### Deployment Timeline
- **Deployed**: October 5, 2025, 15:17 EDT
- **Implementation Time**: ~30 minutes
- **Status**: ✅ Active and Running

## 🔧 Implementation Details

### Build Configuration
- **Go Version**: 1.23.4 (installed at `/usr/local/go`)
- **Binary Location**: `/home/ansible/adguardhome-sync/adguardhome-sync`
- **Build Type**: From source (git repository)
- **Build Command**: `go build -o adguardhome-sync`

### Installation Steps Performed
1. **Go Installation**
   ```bash
   wget --no-check-certificate https://go.dev/dl/go1.23.4.linux-amd64.tar.gz
   sudo tar -C /usr/local -xzf go1.23.4.linux-amd64.tar.gz
   ```

2. **Binary Build**
   ```bash
   export PATH=$PATH:/usr/local/go/bin
   go build -o adguardhome-sync
   ```

3. **Configuration Setup**
   - Created config file at `/home/ansible/.adguardhome-sync.yaml`
   - Configured credentials with special character handling

4. **Service Installation**
   - Created systemd service at `/etc/systemd/system/adguardhome-sync.service`
   - Enabled automatic startup on boot

## ⚙️ Configuration

### Configuration File Location
`/home/ansible/.adguardhome-sync.yaml`

### Active Configuration
```yaml
origin:
  url: http://10.10.20.6:80
  username: admin
  password: "PbHED5b5i@C?"  # Note: Password contains special characters
  apiPath: /control

replicas:
  - url: http://10.10.20.7:80
    username: admin
    password: "PbHED5b5i@C?"
    apiPath: /control

cron: "*/5 * * * *"  # Sync every 5 minutes

api:
  port: 8080  # API server for remote triggering

features:
  generalSettings: true
  queryLogConfig: true
  statsConfig: true
  clientSettings: true
  services: true
  filters: true
  dhcp:
    serverConfig: false  # DHCP not used
    staticLeases: false  # DHCP not used
  dns:
    serverConfig: true
    accessLists: true
    rewrites: true
```

### Sync Schedule
- **Frequency**: Every 5 minutes (`*/5 * * * *`)
- **On Startup**: Yes (immediate sync when service starts)
- **Continue on Error**: Enabled (service won't stop on sync failures)

### Features Synced
✅ **Enabled Features:**
- General Settings
- Query Log Configuration
- Statistics Configuration
- Client Settings
- Services
- Filters (blocklists)
- DNS Server Configuration
- DNS Access Lists
- DNS Rewrites

❌ **Disabled Features:**
- DHCP Server Configuration (not used in environment)
- DHCP Static Leases (not used in environment)
- TLS Configuration

## 🚀 Service Management

### Systemd Service Configuration
**File**: `/etc/systemd/system/adguardhome-sync.service`

```ini
[Unit]
Description=AdGuard Home Sync
After=network.target

[Service]
Type=simple
User=ansible
WorkingDirectory=/home/ansible/adguardhome-sync
Environment="PATH=/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
ExecStart=/home/ansible/adguardhome-sync/adguardhome-sync run --continueOnError
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### Common Service Commands

#### Check Status
```bash
sudo systemctl status adguardhome-sync
```

#### Start/Stop/Restart Service
```bash
sudo systemctl start adguardhome-sync
sudo systemctl stop adguardhome-sync
sudo systemctl restart adguardhome-sync
```

#### View Logs
```bash
# View recent logs
sudo journalctl -u adguardhome-sync -n 50

# Follow logs in real-time
sudo journalctl -u adguardhome-sync -f

# View logs from specific time
sudo journalctl -u adguardhome-sync --since "1 hour ago"
```

#### Disable/Enable Service
```bash
# Disable auto-start
sudo systemctl disable adguardhome-sync

# Enable auto-start
sudo systemctl enable adguardhome-sync
```

## 📊 Monitoring & Troubleshooting

### Health Check Commands

#### Service Health
```bash
# Check if service is running
systemctl is-active adguardhome-sync

# Get detailed service status
systemctl show adguardhome-sync --no-pager
```

#### Test Connectivity
```bash
# Test origin server
curl -u admin:'PbHED5b5i@C?' http://10.10.20.6:80/control/status

# Test replica server
curl -u admin:'PbHED5b5i@C?' http://10.10.20.7:80/control/status
```

#### Manual Sync Trigger
```bash
# Trigger sync via API
curl -X POST http://localhost:8080/api/v1/sync

# Check last sync from logs
sudo journalctl -u adguardhome-sync | grep "Sync done" | tail -1
```

### Common Issues & Solutions

| Issue | Solution |
|-------|----------|
| **Authentication Failed** | Check password in config file, ensure special characters are quoted |
| **Service Won't Start** | Check logs: `sudo journalctl -u adguardhome-sync -n 100` |
| **Sync Not Running** | Verify cron expression in config, check service status |
| **API Not Responding** | Ensure port 8080 is not in use: `sudo netstat -tlnp | grep 8080` |
| **Partial Sync** | Check feature flags in config, review error logs |

### Log Analysis

#### Success Pattern
```
INFO  sync  sync/sync.go:301  Sync done  {"from": "10.10.20.6:80", "to": "10.10.20.7:80"}
```

#### Error Patterns
```bash
# Find authentication errors
sudo journalctl -u adguardhome-sync | grep "401"

# Find connection errors
sudo journalctl -u adguardhome-sync | grep -E "connection refused|timeout"

# Find sync failures
sudo journalctl -u adguardhome-sync | grep ERROR
```

## 🌐 API Endpoints

### Available Endpoints

| Endpoint | Method | Description | Example |
|----------|--------|-------------|---------|
| `/api/v1/sync` | POST | Trigger manual sync | `curl -X POST http://localhost:8080/api/v1/sync` |
| `/metrics` | GET | Prometheus metrics | `curl http://localhost:8080/metrics` |

### API Usage Examples

```bash
# Trigger sync from remote host
curl -X POST http://10.10.20.8:8080/api/v1/sync

# Get Prometheus metrics
curl -s http://localhost:8080/metrics | grep adguardhome

# Check if API is responding
curl -I http://localhost:8080/api/v1/sync 2>/dev/null | head -1
```

## 🛠️ Maintenance

### Update Procedures

#### Update Binary from Source
```bash
# Stop service
sudo systemctl stop adguardhome-sync

# Pull latest changes
cd /home/ansible/adguardhome-sync
git pull

# Rebuild
export PATH=$PATH:/usr/local/go/bin
go build -o adguardhome-sync

# Restart service
sudo systemctl start adguardhome-sync
```

#### Update Configuration
```bash
# Edit config
nano /home/ansible/.adguardhome-sync.yaml

# Restart to apply changes
sudo systemctl restart adguardhome-sync

# Verify changes
sudo journalctl -u adguardhome-sync -f
```

### Backup Procedures

#### Configuration Backup
```bash
# Backup sync configuration
cp /home/ansible/.adguardhome-sync.yaml /home/ansible/.adguardhome-sync.yaml.backup

# Backup AdGuard configurations
ssh 10.10.20.6 "tar -czf /tmp/adguard-primary-backup.tar.gz /opt/AdGuardHome/conf"
ssh 10.10.20.7 "tar -czf /tmp/adguard-replica-backup.tar.gz /opt/AdGuardHome/conf"
```

### Performance Monitoring

```bash
# Check memory usage
systemctl status adguardhome-sync | grep Memory

# Check CPU usage
top -p $(pgrep adguardhome-sync)

# Check sync duration
sudo journalctl -u adguardhome-sync | grep "Sync done" | tail -10
```

## 🔒 Security Considerations

### Current Security Setup

1. **Authentication**
   - Credentials stored in config file with restricted permissions
   - Password contains special characters requiring proper escaping
   - No external API authentication (consider adding if exposing externally)

2. **Network Security**
   - All communication over internal network (10.10.20.0/24)
   - HTTP used (not HTTPS) - acceptable for internal LXC communication
   - API server bound to all interfaces (0.0.0.0:8080)

3. **File Permissions**
   ```bash
   # Config file permissions (should be 600)
   chmod 600 /home/ansible/.adguardhome-sync.yaml

   # Binary permissions
   chmod 750 /home/ansible/adguardhome-sync/adguardhome-sync
   ```

### Security Recommendations

1. **For Production Enhancement**
   - Consider adding API authentication if exposing port 8080 externally
   - Implement HTTPS for API endpoints if accessed over untrusted networks
   - Use secrets management system for passwords instead of plain text config

2. **Access Control**
   ```bash
   # Restrict API access via firewall (if needed)
   sudo ufw allow from 10.10.20.0/24 to any port 8080
   sudo ufw deny 8080
   ```

## 🔄 Recovery Procedures

### Service Recovery

#### If Sync Fails
```bash
# 1. Check logs for errors
sudo journalctl -u adguardhome-sync -n 100 | grep ERROR

# 2. Test connectivity to both servers
curl -u admin:'PbHED5b5i@C?' http://10.10.20.6:80/control/status
curl -u admin:'PbHED5b5i@C?' http://10.10.20.7:80/control/status

# 3. Restart service
sudo systemctl restart adguardhome-sync

# 4. Monitor logs
sudo journalctl -u adguardhome-sync -f
```

#### Complete Service Reinstall
```bash
# Stop and disable service
sudo systemctl stop adguardhome-sync
sudo systemctl disable adguardhome-sync

# Remove service file
sudo rm /etc/systemd/system/adguardhome-sync.service
sudo systemctl daemon-reload

# Rebuild binary
cd /home/ansible/adguardhome-sync
export PATH=$PATH:/usr/local/go/bin
go build -o adguardhome-sync

# Reinstall service
sudo cp /tmp/adguardhome-sync.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable adguardhome-sync
sudo systemctl start adguardhome-sync
```

### Rollback Procedure

#### If Replica Gets Corrupted
```bash
# 1. Stop sync service
sudo systemctl stop adguardhome-sync

# 2. On replica server (10.10.20.7)
ssh 10.10.20.7
systemctl stop AdGuardHome
rm -rf /opt/AdGuardHome/conf
tar -xzf /tmp/adguard-replica-backup.tar.gz -C /
systemctl start AdGuardHome

# 3. Fix sync issues and restart
# Edit config if needed
nano /home/ansible/.adguardhome-sync.yaml

# Restart sync
sudo systemctl start adguardhome-sync
```

## 📈 Performance Metrics

### Current Performance
- **Sync Duration**: ~1.4 seconds per sync
- **Memory Usage**: ~6.3 MB
- **CPU Usage**: Minimal (<1% average)
- **Network Traffic**: Minimal (JSON payloads only)
- **Sync Frequency**: 288 syncs per day (every 5 minutes)

### Resource Impact
- **Disk Space**: ~36 MB for binary, minimal for config
- **Network Load**: Negligible on internal network
- **System Load**: Minimal impact on LXC container

## 📝 Additional Notes

### Special Considerations
1. **Password Handling**: The password contains a `?` character which requires proper escaping in YAML
2. **Go Binary**: Built from source, allowing for easy updates and customization
3. **Cron vs Systemd Timer**: Using cron expression within the app instead of systemd timer for simplicity
4. **Error Handling**: `continueOnError` flag ensures service stability even if individual syncs fail

## 🎨 UI Enhancements

### Dark Mode Toggle
**Implemented**: October 5, 2025

#### Overview
Added a client-side dark mode toggle to the web UI allowing users to switch between light and dark themes with persistent preference storage.

#### Implementation Details

**Files Modified**:
1. **`/home/ansible/adguardhome-sync/internal/sync/static/static.go`**
   - Exposed both theme CSS files at separate endpoints
   - Added `/lib/bootstrap-light.css` endpoint
   - Added `/lib/bootstrap-dark.css` endpoint
   - Maintained backward compatibility with `/lib/bootstrap.css`

2. **`/home/ansible/adguardhome-sync/internal/sync/static/index.html`**
   - Added theme toggle button in header (top-right corner)
   - Implemented JavaScript theme switching logic
   - Added localStorage persistence for theme preference
   - Icon changes: 🌙 (light mode) ↔️ ☀️ (dark mode)

#### Features
- **Persistent Storage**: Theme preference saved to browser localStorage
- **Automatic Loading**: Saved theme applied on page load
- **Seamless Switching**: No page reload required
- **Bootstrap Themes**:
  - Light: Bootstrap 5.3.3 default
  - Dark: Bootswatch Darkly 5.3 theme

#### Code Changes

**Theme Toggle Button**:
```html
<button id="theme-toggle" class="btn btn-outline-secondary" title="Toggle dark mode">
    <span id="theme-icon">🌙</span>
</button>
```

**JavaScript Logic**:
```javascript
function setTheme(isDark) {
    if (isDark) {
        themeStylesheet.href = 'lib/bootstrap-dark.css';
        themeIcon.textContent = '☀️';
        localStorage.setItem('theme', 'dark');
    } else {
        themeStylesheet.href = 'lib/bootstrap-light.css';
        themeIcon.textContent = '🌙';
        localStorage.setItem('theme', 'light');
    }
}
```

#### Deployment Steps
```bash
# 1. Modified source files (static.go and index.html)

# 2. Rebuild binary
/usr/local/go/bin/go build -o adguardhome-sync

# 3. Restart service to load new binary
sudo systemctl restart adguardhome-sync

# 4. Verify deployment
curl -s http://localhost:8080/ | grep "theme-toggle"
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/lib/bootstrap-light.css
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/lib/bootstrap-dark.css
```

#### User Experience
- Toggle appears in top-right corner of header
- Single click switches theme instantly
- Preference persists across browser sessions and page refreshes
- No additional dependencies required (uses existing Bootstrap themes)

#### Technical Notes
- Leverages existing dark theme CSS already bundled with application
- Client-side implementation - no server-side changes to theme handling
- Backward compatible - server-side dark mode config still works as default
- No external dependencies or frameworks added
- Lightweight implementation (~30 lines of JavaScript)

### Sync Progress Indicator
**Implemented**: October 5, 2025

#### Overview
Added a real-time sync progress indicator to the web UI displaying countdown timers, progress bar, and sync status without requiring page refreshes.

#### Implementation Details

**Backend Changes**:

1. **`/home/ansible/adguardhome-sync/internal/sync/sync.go`**
   - Added time tracking fields to worker struct:
     - `lastSyncTime time.Time` - Tracks when last sync completed
     - `nextSyncTime time.Time` - Tracks when next sync is scheduled
     - `cronEntryID cron.EntryID` - Stores cron entry for schedule inspection
   - Updated `sync()` to record completion time and calculate next sync time

2. **`/home/ansible/adguardhome-sync/internal/sync/http.go`**
   - Created new API endpoint: `GET /api/v1/sync-schedule`
   - Returns JSON with sync timing information:
     ```json
     {
       "lastSyncTime": "2025-10-05T17:07:37.726Z",
       "nextSyncTime": "2025-10-05T17:10:00.000Z",
       "syncRunning": false,
       "cronExpression": "*/5 * * * *",
       "intervalSeconds": 300
     }
     ```
   - Dynamically calculates interval from cron schedule (works with any cron expression)

**Frontend Changes**:

3. **`/home/ansible/adguardhome-sync/internal/sync/static/index.html`**
   - Added progress indicator HTML between control buttons and logs
   - Implemented real-time countdown with 1-second updates
   - Added visual progress bar showing elapsed time vs total interval
   - Status indicator with animated dot (green when idle, pulsing blue when syncing)

#### Features
- **Real-time Countdown**: Shows "Last sync: Xm Ys ago | Next sync: in Xm Ys"
- **Visual Progress Bar**: Fills from 0-100% between sync intervals
- **Status Indicator**: Animated dot shows sync status
- **Theme-Aware Styling**: Automatically adapts to light/dark mode
- **Automatic Updates**: Polls server every 30s, updates UI every 1s
- **Dynamic Interval**: Works with any cron expression (5min, 10min, hourly, etc.)

#### Code Implementation

**HTML Structure**:
```html
<div class="sync-progress-card">
    <div class="progress-header">
        <span class="sync-status">
            <span class="status-dot"></span>
            <span id="syncStatusText">Idle</span>
        </span>
        <span class="sync-timing">
            Last sync: <span id="lastSyncTime">--</span> ago |
            Next sync: in <span id="nextSyncTime">--</span>
        </span>
    </div>
    <div class="progress-bar-container">
        <div class="progress-bar-fill" id="progressBarFill"></div>
    </div>
</div>
```

**JavaScript Logic**:
```javascript
// Fetch sync schedule every 30 seconds
function fetchSyncSchedule() {
    $.get("api/v1/sync-schedule", function(data) {
        syncScheduleData = data;
        if (data.intervalSeconds > 0 || data.cronExpression) {
            $("#syncProgressRow").show();
        }
    });
}

// Update UI every second
function updateProgressDisplay() {
    const timeSinceSync = Math.floor((Date.now() - lastSync) / 1000);
    const progress = (timeSinceSync / totalInterval) * 100;
    $("#progressBarFill").css("width", progress + "%");
}
```

**CSS Styling** (Theme-Aware):
```css
/* Light mode */
body.theme-light .sync-progress-card {
    background: rgba(0, 0, 0, 0.03);
    border: 1px solid rgba(0, 0, 0, 0.1);
}

/* Dark mode */
body.theme-dark .sync-progress-card {
    background: rgba(0, 0, 0, 0.2);
    border: 1px solid rgba(255, 255, 255, 0.1);
}

/* Layout stability fix */
html {
    overflow-y: scroll;
    scrollbar-gutter: stable;
}

h1 {
    font-size: 2rem !important;
    line-height: 1.2 !important;
}
```

#### Deployment Steps
```bash
# 1. Modified source files
#    - internal/sync/sync.go (backend tracking)
#    - internal/sync/http.go (API endpoint)
#    - internal/sync/static/index.html (UI)

# 2. Rebuild binary
/usr/local/go/bin/go build -o adguardhome-sync

# 3. Restart service
sudo systemctl restart adguardhome-sync

# 4. Verify deployment
curl http://localhost:8080/api/v1/sync-schedule | jq
```

#### User Experience
- Progress indicator appears automatically if cron schedule is configured
- Real-time countdown updates every second without page refresh
- Visual progress bar provides intuitive sync cycle feedback
- Works seamlessly with theme toggle (no layout shifts)
- Shows "Syncing..." status during active sync operations

#### Technical Notes
- **No External Dependencies**: Uses existing jQuery and Bootstrap
- **Minimal Performance Impact**: API polled every 30s, UI updates every 1s
- **Cron-Agnostic**: Automatically calculates interval from any valid cron expression
- **Layout Stability**: Fixed Bootstrap theme CSS differences causing button shifts
- **Responsive Design**: Works on all screen sizes

#### Troubleshooting
If progress indicator doesn't appear:
```bash
# Check API endpoint
curl http://localhost:8080/api/v1/sync-schedule

# Should return sync schedule data, not 404
# If 404, rebuild and restart service
```

### Future Enhancements
- [ ] Add Prometheus metrics collection and Grafana dashboard
- [ ] Implement webhook notifications for sync failures
- [ ] Add health check endpoint for monitoring systems
- [ ] Consider implementing HTTPS for API endpoints
- [ ] Add automated backup before each sync
- [ ] Implement sync history and statistics tracking
- [x] Dark mode toggle for web UI ✅ *Completed October 5, 2025*
- [x] Sync progress indicator ✅ *Completed October 5, 2025*

### Support Information
- **Project Repository**: https://github.com/bakito/adguardhome-sync
- **Documentation**: See [CLAUDE.md](../CLAUDE.md) for development guidance
- **Implementation Plan**: See [IMPLEMENTATION_PLAN.md](./IMPLEMENTATION_PLAN.md) for original planning document

---

*Last Updated: October 5, 2025*
*Implemented by: Claude Code Assistant*
*Environment: Proxmox LXC Containers*