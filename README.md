# Proxmox VM Rollback Timer

This setup rolls back a Proxmox VM to a snapshot every 3 days using a systemd timer.

## Files

- `/root/daily-vm-rollback.sh`  
  Rollback script that stops the VM, rolls it back to a snapshot, and starts it again.

- `/etc/systemd/system/vm-rollback.service`  
  systemd oneshot service that runs the rollback script.

- `/etc/systemd/system/vm-rollback.timer`  
  systemd timer that triggers the service every 3 days.

## Rollback script

Example:

```bash
#!/usr/bin/env bash
set -euo pipefail

VMID="123"
SNAP="daily-reset"

qm stop "$VMID" --skiplock 1 || true
qm rollback "$VMID" "$SNAP"
qm start "$VMID"
```

## Service file

`/etc/systemd/system/vm-rollback.service`

```ini
[Unit]
Description=Rollback Proxmox VM to snapshot

[Service]
Type=oneshot
ExecStart=/root/daily-vm-rollback.sh
```

## Timer file

`/etc/systemd/system/vm-rollback.timer`

```ini
[Unit]
Description=Run Proxmox VM rollback every 3 days

[Timer]
OnBootSec=10m
OnUnitActiveSec=3d
Persistent=true

[Install]
WantedBy=timers.target
```

## Enable the timer

```bash
systemctl daemon-reload
systemctl enable --now vm-rollback.timer
```

## Check status

```bash
systemctl list-timers vm-rollback.timer
systemctl status vm-rollback.timer
journalctl -u vm-rollback.service
```

## Manual test

```bash
systemctl start vm-rollback.service
```

## Notes

- Replace `123` with your actual VM ID.
- Replace `daily-reset` with your snapshot name.
- `Persistent=true` means missed runs will execute after reboot.
- This runs every 3 days relative to the last run, not at a fixed wall-clock date/time.
- Snapshots are not backups.
