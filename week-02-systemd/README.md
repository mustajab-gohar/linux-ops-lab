# Week 02: Systemd & Operational Logging

## Scenario
Managing custom background processes and ensuring they survive reboots is a core operational requirement. This lab demonstrates how to deploy a custom application (a Bash script) as a `systemd` daemon, enforce least-privilege execution, and natively aggregate standard output/error logs via `journald`.

## Executed Tasks

### 1. Application Provisioning
Isolated the application script into a standard operational directory (`/opt`) and enforced execution permissions.
```bash
sudo mkdir -p /opt/custom_services
sudo cp logtime.sh /opt/custom_services/logtime.sh
sudo chmod +x /opt/custom_services/logtime.sh
```

### 2. Least Privilege Execution (IAM)
Running services as `root` introduces severe security vulnerabilities if the application is compromised. A dedicated, shell-less system user was created strictly to execute this specific service.
```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin logtime
```

### 3. Daemon Configuration & Deployment
Installed the custom unit file (`logtime.service`) into the systemd hierarchy, reloaded the daemon to register the new configuration, and enabled the service to start immediately and persist across reboots.
```bash
sudo cp logtime.service /etc/systemd/system/logtime.service
sudo systemctl daemon-reload
sudo systemctl enable --now logtime.service
```

## Verification & Troubleshooting
Bypassing manual log files, the service output is managed directly by the system journal. The following runbook commands are used to verify uptime and audit logs.
```bash
# Check the current active state of the daemon
systemctl status logtime.service --no-pager

# Tail the live application logs in real-time
journalctl -u logtime.service -f --no-pager

# Audit historical logs for recent anomalies
journalctl -u logtime.service --since "10 minutes ago" --no-pager
```

## Operational Hardening Notes
The `logtime.service` unit file was designed with the following security boundaries:

- User Isolation: Executes strictly as the `logtime` system user.
- Native Logging: Prevents disk-fill issues by relying on `journald` log rotation rather than writing to unstructured text files.
- Sandboxing Readiness: The architecture supports strict systemd drop-in directives for future hardening, such as `NoNewPrivileges=true`, `PrivateTmp=true`, `ProtectSystem=full`, and `ProtectHome=true`.
