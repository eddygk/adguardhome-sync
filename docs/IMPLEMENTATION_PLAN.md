# AdGuard Home Sync Implementation Plan

## Environment Overview

| Server | IP Address | Role | Description |
|--------|------------|------|-------------|
| Server 1 | 10.10.20.6 | Origin (Primary) | AdGuard Home primary instance |
| Server 2 | 10.10.20.7 | Replica | AdGuard Home replica instance |
| Sync Server | 10.10.20.8 | Sync Controller | Current server running adguardhome-sync |

All servers are Proxmox LXC containers on the same host.

## Implementation Strategy

### Phase 1: Prerequisites & Information Gathering

**Required Information from You:**
1. **AdGuard Home Credentials**
   - Server 1 (10.10.20.6): Username and password for admin access
   - Server 2 (10.10.20.7): Username and password for admin access

2. **AdGuard Home API Ports** (typically 3000 or 80)
   - Server 1 API port
   - Server 2 API port

3. **Sync Requirements**
   - Which features to sync (filters, rewrites, DNS settings, DHCP, clients, etc.)
   - Sync frequency (e.g., every 5 minutes, hourly, etc.)
   - Whether to run as daemon or one-time sync

4. **Security Preferences**
   - Use HTTPS or HTTP for API connections
   - Whether to enable API server for remote triggering
   - Preferred API server port (default 8080)

### Phase 2: Configuration Setup

#### Option A: Environment Variables Configuration
Create a systemd service with environment variables:

```bash
# /etc/systemd/system/adguardhome-sync.service
[Unit]
Description=AdGuard Home Sync
After=network.target

[Service]
Type=simple
User=ansible
WorkingDirectory=/home/ansible/adguardhome-sync
Environment="LOG_LEVEL=info"
Environment="LOG_FORMAT=console"
Environment="ORIGIN_URL=http://10.10.20.6:3000"
Environment="ORIGIN_USERNAME=<username>"
Environment="ORIGIN_PASSWORD=<password>"
Environment="REPLICA1_URL=http://10.10.20.7:3000"
Environment="REPLICA1_USERNAME=<username>"
Environment="REPLICA1_PASSWORD=<password>"
Environment="CRON=0 */5 * * *"  # Every 5 minutes
Environment="API_PORT=8080"
Environment="FEATURES_GENERALSETTINGS=true"
Environment="FEATURES_QUERYLOGCONFIG=true"
Environment="FEATURES_STATSCONFIG=true"
Environment="FEATURES_CLIENTSETTINGS=true"
Environment="FEATURES_SERVICES=true"
Environment="FEATURES_FILTERS=true"
Environment="FEATURES_DHCP_SERVERCONFIG=false"
Environment="FEATURES_DHCP_STATICLEASES=false"
Environment="FEATURES_DNS_SERVERCONFIG=true"
Environment="FEATURES_DNS_ACCESSLISTS=true"
Environment="FEATURES_DNS_REWRITES=true"
ExecStart=/home/ansible/adguardhome-sync/adguardhome-sync run
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

#### Option B: YAML Configuration File
Create configuration file at `/home/ansible/.adguardhome-sync.yaml`:

```yaml
origin:
  url: http://10.10.20.6:3000
  username: <username>
  password: <password>
  apiPath: /control
  insecureSkipVerify: false

replicas:
  - url: http://10.10.20.7:3000
    username: <username>
    password: <password>
    apiPath: /control
    insecureSkipVerify: false
    autoSetup: false

cron: "0 */5 * * *"  # Every 5 minutes

api:
  port: 8080
  username: ""  # Optional API authentication
  password: ""

features:
  generalSettings: true
  queryLogConfig: true
  statsConfig: true
  clientSettings: true
  services: true
  filters: true
  dhcp:
    serverConfig: false
    staticLeases: false
  dns:
    serverConfig: true
    accessLists: true
    rewrites: true
```

### Phase 3: Testing Strategy

1. **Initial Connection Test**
   ```bash
   # Test connectivity to both AdGuard instances
   curl -u <username>:<password> http://10.10.20.6:3000/control/status
   curl -u <username>:<password> http://10.10.20.7:3000/control/status
   ```

2. **Dry Run (One-time sync)**
   ```bash
   # Build the binary
   go build -o adguardhome-sync

   # Test with explicit parameters (no daemon mode)
   ./adguardhome-sync run \
     --origin-url http://10.10.20.6:3000 \
     --origin-username <username> \
     --origin-password <password> \
     --replica-url http://10.10.20.7:3000 \
     --replica-username <username> \
     --replica-password <password> \
     --continue-on-error
   ```

3. **Feature-by-Feature Testing**
   - Start with non-critical features (filters, rewrites)
   - Gradually enable more features
   - Monitor logs for errors

4. **Verification Steps**
   - Check Server 2 UI for synced configurations
   - Verify filters are present
   - Check DNS rewrites
   - Validate client settings

### Phase 4: Deployment Options

#### Option 1: Systemd Service (Recommended)
```bash
# Copy binary
sudo cp adguardhome-sync /usr/local/bin/

# Create service file (as shown above)
sudo systemctl daemon-reload
sudo systemctl enable adguardhome-sync
sudo systemctl start adguardhome-sync

# Monitor logs
sudo journalctl -u adguardhome-sync -f
```

#### Option 2: Docker Container
```bash
# Create docker-compose.yml
cat > docker-compose.yml <<EOF
version: '3'
services:
  adguardhome-sync:
    image: ghcr.io/bakito/adguardhome-sync:latest
    container_name: adguardhome-sync
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - LOG_LEVEL=info
      - ORIGIN_URL=http://10.10.20.6:3000
      - ORIGIN_USERNAME=<username>
      - ORIGIN_PASSWORD=<password>
      - REPLICA1_URL=http://10.10.20.7:3000
      - REPLICA1_USERNAME=<username>
      - REPLICA1_PASSWORD=<password>
      - CRON=0 */5 * * *
EOF

docker-compose up -d
```

#### Option 3: Cron Job (Simple)
```bash
# Add to crontab
*/5 * * * * /home/ansible/adguardhome-sync/adguardhome-sync run >> /var/log/adguardhome-sync.log 2>&1
```

### Phase 5: Monitoring & Maintenance

1. **API Endpoints for Monitoring**
   - Health check: `http://10.10.20.8:8080/health`
   - Trigger sync: `POST http://10.10.20.8:8080/api/v1/sync`
   - Prometheus metrics: `http://10.10.20.8:8080/metrics`

2. **Log Monitoring**
   ```bash
   # For systemd
   journalctl -u adguardhome-sync -f

   # For docker
   docker logs -f adguardhome-sync
   ```

3. **Alerting Setup** (Optional)
   - Configure alerts for sync failures
   - Monitor API endpoint availability
   - Track sync duration metrics

### Phase 6: Backup & Recovery

1. **Before First Sync**
   - Backup Server 2 configuration
   ```bash
   # On Server 2 (10.10.20.7)
   cp -r /opt/AdGuardHome/conf /opt/AdGuardHome/conf.backup
   ```

2. **Recovery Procedure**
   - If sync causes issues, restore backup
   - Disable sync service
   - Review logs for root cause

## Security Considerations

1. **Credential Storage**
   - Use environment variables or secure config file
   - Set appropriate file permissions (600)
   - Consider using secrets management for production

2. **Network Security**
   - Since all containers are on same host, internal network is secure
   - Consider HTTPS if exposing API externally
   - Implement API authentication if needed

3. **Access Control**
   - Limit sync server access to necessary personnel
   - Use firewall rules if needed
   - Monitor access logs

## Rollback Plan

If issues occur:
1. Stop sync service: `sudo systemctl stop adguardhome-sync`
2. Restore Server 2 backup configuration
3. Restart AdGuard Home on Server 2
4. Review logs to identify issue
5. Adjust configuration and test again

## Success Criteria

- [ ] Successful initial sync without errors
- [ ] All selected features properly synchronized
- [ ] Automated sync running on schedule
- [ ] API endpoints accessible for monitoring
- [ ] No performance impact on AdGuard services
- [ ] Logs show consistent successful syncs

## Next Steps

Once you provide the required information:
1. Configure authentication credentials
2. Build the binary
3. Run initial test sync
4. Set up automated sync service
5. Configure monitoring
6. Document final configuration

## Required Information Summary

Please provide:
1. **AdGuard Home admin credentials** for both servers (10.10.20.6 and 10.10.20.7)
2. **API ports** for both AdGuard instances (usually 3000 or 80)
3. **Features to sync** (filters, DNS settings, clients, etc.)
4. **Sync frequency** preference (every 5 min, hourly, etc.)
5. **Deployment preference** (systemd service, Docker, or cron)
6. **API server requirements** (port, authentication needs)

With this information, we can proceed with the implementation.