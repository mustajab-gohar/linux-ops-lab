# Week 02 — systemd: logtime.service

A simple systemd service running a Bash script and logging to journald.

## Files
- `logtime.sh`
- `logtime.service`

## Install (on the VM)
### 1) Put the script in place
```bash
sudo mkdir -p /opt/custom_services
sudo cp logtime.sh /opt/custom_services/logtime.sh
sudo chmod +x /opt/custom_services/logtime.sh
```

### 2) Create a dedicated system user (least privilege)
Run the service as a non-root user to reduce risk if the service is compromised.

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin logtime

```

### 3) Install the unit file
```bash
sudo cp logtime.service /etc/systemd/system/logtime.service
sudo systemctl daemon-reload
sudo systemctl enable --now logtime.service
```

### 4) Verify
```bash
systemctl status logtime.service --no-pager
journalctl -u logtime.service -f --no-pager

```

### 5) Troubleshooting
```bash
systemctl status logtime.service --no-pager
journalctl -u logtime.service --since "10 minutes ago" --no-pager
```

## Hardening Notes
- Runs as a dedicated user (non-root)
- Logs to journald (stdout/stderr)
- Optional sandboxing: NoNewPrivileges, PrivateTmp, ProtectSystem, ProtectHome

