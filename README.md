# Proxmox VM Rollback Timer

This project rolls back a Proxmox VM to a snapshot every 3 days using a systemd service and timer.

## Files

- `daily-vm-rollback.sh`  
  Rollback script that stops the VM, rolls it back to a snapshot, and starts it again.

- `vm-rollback.service`  
  systemd oneshot service that runs the rollback script.

- `vm-rollback.timer`  
  systemd timer that triggers the service every 3 days.

## Prerequisites

- A Proxmox VE host
- A VM with an existing snapshot to roll back to
- Root or sudo access on the Proxmox host
- The `qm` command available on the host

## 1. Create a snapshot

In the Proxmox web UI:

1. Select your VM
2. Open **Snapshots**
3. Click **Take Snapshot**
4. Name it something like `daily-reset`

You can also create one from the shell:

```bash
qm snapshot 123 daily-reset
```

Replace `123` with your VM ID.

## 2. Configure the rollback script

Edit `daily-vm-rollback.sh` and set your VM ID and snapshot name:

```bash
#!/usr/bin/env bash
set -euo pipefail

VMID="123"
SNAP="daily-reset"

qm stop "$VMID" --skiplock 1 || true
qm rollback "$VMID" "$SNAP"
qm start "$VMID"
```

## 3. Install the files on the Proxmox host

Copy the files to the correct locations:

```bash
install -m 0755 daily-vm-rollback.sh /root/daily-vm-rollback.sh
install -m 0644 vm-rollback.service /etc/systemd/system/vm-rollback.service
install -m 0644 vm-rollback.timer /etc/systemd/system/vm-rollback.timer
```

## 4. Enable the timer

Reload systemd and start the timer:

```bash
systemctl daemon-reload
systemctl enable --now vm-rollback.timer
```

## 5. Check status

See when the timer will run next:

```bash
systemctl list-timers vm-rollback.timer
```

Check timer status:

```bash
systemctl status vm-rollback.timer
```

Check service logs:

```bash
journalctl -u vm-rollback.service
```

## 6. Test manually

Run the rollback once manually:

```bash
systemctl start vm-rollback.service
```

Then confirm the VM stopped, rolled back, and started again.

## How the timer works

The timer is configured like this:

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

### Behavior

- `OnBootSec=10m`  
  Runs the first time 10 minutes after boot.

- `OnUnitActiveSec=3d`  
  Runs again every 3 days after the last successful service run.

- `Persistent=true`  
  If the host is off when a run is due, systemd will trigger it after boot.

## Notes

- Replace `123` with your actual VM ID.
- Replace `daily-reset` with your actual snapshot name.
- This schedule is based on elapsed time, not a fixed calendar date.
- Rolling back a VM snapshot is not the same as making a backup.
- Stopping the VM before rollback is safer for consistency.

## Example full setup

```bash
chmod +x daily-vm-rollback.sh
install -m 0755 daily-vm-rollback.sh /root/daily-vm-rollback.sh
install -m 0644 vm-rollback.service /etc/systemd/system/vm-rollback.service
install -m 0644 vm-rollback.timer /etc/systemd/system/vm-rollback.timer
systemctl daemon-reload
systemctl enable --now vm-rollback.timer
systemctl list-timers vm-rollback.timer
```

## Optional improvement

If you want a cleaner name, you may want to rename `daily-vm-rollback.sh` to `vm-rollback.sh`, since it now runs every 3 days rather than daily.
