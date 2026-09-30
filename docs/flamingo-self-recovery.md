# Flamingo: Self-Recovery Plan

Carried over from a Claude Web session after an outage that stacked four problems: a failed resource
start that stayed down for 11 hours, no alerting, the flex share silently hidden (writes went to the
underlying disk), and clock drift.

Nodes: white (primary storage), green (secondary), red, blue. NFS/SMB and CTs 100/101 are managed by
Pacemaker (`grp_storage`) and Proxmox HA. lsyncd replicates white → green with `--delete`.

## 1. Failback is the biggest data-loss risk
Scenario: white dies → services fail over to green, users write to green → white returns →
Pacemaker (white score 100, no stickiness) and HA (`failback` on by default) move services back to
white → lsyncd starts at boot on white and pushes white's **stale** tree to green with `--delete`,
wiping everything written on green.

- [ ] **Stop automatic failback (cheap, do now):**
  ```bash
  pcs resource defaults update resource-stickiness=200
  ha-manager set ct:100 --failback 0
  ha-manager set ct:101 --failback 0
  ```
  Moving back becomes deliberate: stop services on green, rsync green → white, then move.
- [ ] **Run lsyncd under Pacemaker** (in `grp_storage`, a per-node config pointing at the peer), so
  only the active node pushes, toward the standby. Needs a real failover test. ⚠️ Restarting lsyncd
  triggers a full ~1.2TB sweep.

## 2. Let Pacemaker retry failed resources
- [ ] `pcs resource defaults update failure-timeout=10min migration-threshold=3`
- [ ] Investigate why `grp_storage` didn't fail over to green after white's start failure:
  `crm_simulate -sL`.

## 3. Make missing mounts fail loudly
Mark the underlying mountpoint directories immutable so writes fail when nothing is mounted:
```bash
# On white:
mkdir -p /mnt/under
mount --bind / /mnt/under              # root fs without submounts
chattr +i /mnt/under/srv/shared
umount /mnt/under
mount --bind /srv/shared /mnt/under    # data fs without flex on top
chattr +i /mnt/under/flex
umount /mnt/under
# On green: only the /srv/shared half (also protects green's ~10GB root from a 1.2TB lsyncd push)
```
- [ ] white  - [ ] green

## 4. Alerting
- [ ] systemd timer on red, every 5 min: VIP + NFS read, SMB on .128, lsyncd liveness + test-file
  round trip, flex and data mountpoints, `timedatectl` synchronized on all four nodes, HA LRM
  freshness. Push failures via ntfy or email (PVE's postfix/notification system).

## 5. Power
- [ ] UPS + NUT for coordinated shutdown (see `ups-power-budget.md`).
- [ ] BIOS "restore on AC power loss" set consistently on all four nodes ("last state" or "on").
- [ ] Open question: why the network didn't come up after the reboot, and what was fixed manually.

## 6. Guardrails
- [ ] `/etc/motd` on white and green: "NFS, VIP, and CT 100/101 are cluster-managed; use
  `pcs resource cleanup`, not systemctl".
- [ ] Replace lsyncd's SysV script (reports "active (exited)" even when dead) with a native systemd
  unit (`Restart=on-failure`). This fits with the Pacemaker move in #1.

## Suggested order
1. #1 part 1 (stickiness/failback) and #2: one-liners, no disruption.
2. #3 and the monitoring in #4.
3. lsyncd under Pacemaker + a real failover test, when there's time for the sweep.
