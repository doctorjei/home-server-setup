# mantis: NVMe Drive Health Record

Baseline and diagnostic results for mantis's boot drive, recorded during the cluster rebuild
while troubleshooting a boot failure. Checks were run from a TinyCore live environment.

## Drive
| Field            | Value                         |
|--                |--                             |
| Model            | Crucial P1 1TB (`CT1000P1SSD8`), QLC |
| Serial           | `1921E20664C3`                |
| Firmware         | `P3CR010` (original; `P3CR013` / `P3CR021` exist) |
| Device           | `/dev/nvme0n1`                |

## Partition Layout
| Partition        | Type   | Size     | Notes                                           |
|--                |--      |--        |--                                               |
| `nvme0n1p1`      | FAT32  | 1 GiB    | ESP; TinyCore's `tce/` extension dir also lives here |
| `nvme0n1p2`      | ext4   | 1.5 GiB  | Likely `/boot`                                  |
| `nvme0n1p3`      | ext4   | 4 GiB    | ~75% full; purpose TBD                          |
| `nvme0n1p4`      | btrfs  | ~925 GiB | Subvolumes `@` (root) and `@home`; created 2026-03-27 |
| `nvme0n1p7`      | ext4   | 12 MiB   | Nearly empty; purpose TBD                       |

## SMART Baseline (2026-09-30)
| Attribute                        | Value    |
|--                                |--        |
| Overall health                   | PASSED   |
| Critical Warning                 | `0x00`   |
| Percentage Used                  | 2%       |
| Available Spare                  | 100%     |
| Power On Hours                   | 46,769   |
| Unsafe Shutdowns                 | 68       |
| **Media and Data Integrity Errors** | **177** |
| Error Information Log Entries    | 0        |

The 177 media errors did not increase after two full reads of the drive. The btrfs filesystem is
~4,400 hours old (about 10% of the drive's life), and btrfs's own error counters are all zero, so
the errors most likely predate the current install.

## Diagnostic Results (2026-09-30)
| Check                                         | Result                                   |
|--                                             |--                                        |
| Full-disk read (`dd`, 1.0 TB @ 1.9 GB/s)      | Clean, no I/O errors                     |
| ESP (`fsck.vfat`)                             | Dirty bit set; cleared with `-a -w`, now clean |
| ext4 p2 / p3 / p7 (`e2fsck -fn`)              | Clean                                    |
| btrfs structure (`btrfs check --readonly`)    | No errors (~227 GB used)                 |
| btrfs device stats                            | All counters 0                           |
| btrfs scrub (216 GiB)                         | No errors                                |
| `/var/log/journal` (1.3 GB)                   | Lists and reads without stalling         |

**Conclusion:** the stored data and all filesystems are healthy. The journal was last written at
14:11 on 2026-09-30, so at least one recent boot got as far as systemd.

## Controller Lockup (2026-09-30), Likely Cause of Boot Failure
Running `journalctl -D /var/log/journal --list-boots` (chrooted into `@`) hung in `D` state, and
the whole system froze. The kernel log showed:

```
nvme nvme0: I/O tag 70 (e046) opcode 0x2 (I/O Cmd) QID 6 timeout, aborting req_op:READ(0) size:131072
  (x4, tags 70-73)
nvme nvme0: I/O tag 70 (e046) opcode 0x2 (I/O Cmd) QID 6 timeout, reset controller
nvme nvme0: Device not ready; aborting reset, CSTS=0x1
nvme nvme0: Abort status: 0x371
nvme0n1: I/O Cmd(0x2) @ LBA 454954056, 256 blocks, I/O Error (sct 0x3 / sc 0x71)
I/O error, dev nvme0n1, sector 454953800 op 0x0:(READ) ...
```

Interpretation:
- `sct 0x3 / sc 0x71` / `0x371` = *command aborted by host*: Linux cancelled the requests after
  timeouts. The drive did not report media errors, and these sectors (~233 GB in, inside p4) read
  cleanly during the full `dd` scan.
- `CSTS=0x1`: the controller is still on the PCIe bus and claims ready, but ignores commands,
  including reset. This is a **firmware lockup**, not a PCIe dropout (that would be `CSTS=0xffffffff`).
- Long sequential reads (`dd`, scrub) pass; bursty small reads (`ls`, `journalctl`) trigger the hang.
- Recovery requires a full power cycle (unplug ~10 s).

Suspects, in order: old firmware `P3CR010`; APST (autonomous power-state transitions) triggering
the firmware bug.

### Resolution: APST Disabled (2026-09-30)
- With `nvme_core.default_ps_max_latency_us=0`, the same `journalctl --list-boots` that froze the
  system completed in 2 s.
- Journal history shows failed boots since at least 2026-09-28: most stop within seconds of
  `Flush Journal to Persistent Storage` (first disk write burst); the longest (09:29 on 09-30) ran a
  normal desktop session for 6 minutes, then logging stopped mid-activity. No kernel or relevant
  package changes since the March 2026 install (kernel `6.17.0-19`), so the drive's behavior changed
  on its own (aging QLC + original firmware).
- Fix made permanent: `nvme_core.default_ps_max_latency_us=0` added to `GRUB_CMDLINE_LINUX_DEFAULT`
  in `/etc/default/grub` (backup at `/etc/default/grub.bak`), then `update-grub`. mantis now boots
  normally and is reachable over SSH (`himawari@192.168.2.32`).

### Remaining
**Status (2026-09-30): working.** mantis boots unattended, is reachable over SSH, and the display works in 4K@60. An OS reinstall
is planned. On reinstall, the APST fix is lost with the old GRUB config: either update the drive
firmware first, or add `nvme_core.default_ps_max_latency_us=0` at the installer's GRUB prompt
(press `e`) **and** to `/etc/default/grub` on the new system before its first normal boot.
Otherwise the installer or new OS may lock up the same way.

1. Update firmware to `P3CR021` via Crucial's bootable ISO (back up first). Keep the APST
   workaround until then.
2. Watch the media error counter (baseline 177); budget for a replacement drive.
3. Separate issue, **no HDMI output: fixed (4K@60).** The display is a Vizio E70-E3 4K TV on
   `HDMI-0` (RTX 4070, driver 580.126.09). It no longer shows the 30 Hz modes on this input (it did
   in March), but `3840x2160 @ 59.94` works. X started at 59.94, but LightDM autologs in as
   `himawari`, and the Cinnamon session then switched to the TV's preferred `3840x2160 @ 30`
   (blank), even with `cinnamon-monitors.xml` set to `59.940`. Current setup (all three layers):
   - `/etc/X11/xorg.conf.d/20-nvidia-hdmi.conf`: `Option "metamodes" "HDMI-0: 3840x2160_60 +0+0"`
   - `/usr/local/bin/fix-hdmi-mode` (runs `/sbin/prime-offload`, then
     `xrandr --output HDMI-0 --mode 3840x2160 --rate 59.94`), hooked in
     `/etc/lightdm/lightdm.conf.d/90-hdmi-mode.conf` as `display-setup-script` and
     `session-setup-script`
   - `~/.config/autostart/fix-hdmi-mode.desktop`: runs the same `xrandr` 5 s after login. **This
     is the part that actually fixes it**, because it runs after Cinnamon. Expect a few seconds of
     black after the splash.
   - Emergency picture over SSH:
     `sudo DISPLAY=:0 XAUTHORITY=/var/run/lightdm/root/:0 xrandr --output HDMI-0 --mode 3840x2160 --rate 59.94`
   - Mode tester: `~/modetest.sh` (tries each mode, asks y/n, auto-reverts after 30 s).
4. Cleanup: `casper-md5check.service` fails on every boot (installer leftover). Remove with
   `sudo apt purge casper`.

## Monitoring
Re-check periodically. If the media error count rises above 177, plan to replace the drive.

```bash
sudo smartctl -a /dev/nvme0n1 | grep -E 'Media|Unsafe|Percentage'
sudo btrfs device stats /
```

## TinyCore Notes
- BusyBox `dd` lacks `status=progress`; install `coreutils` and use `/usr/local/bin/dd`.
  Don't pipe it into `tail`, which hides the progress output.
- btrfs: `tce-load -wi btrfs-progs`, then `sudo modprobe btrfs` before mounting, and pass
  `-t btrfs` explicitly (BusyBox `mount` otherwise fails with "Invalid argument").
- Avoid running `fsck` on p1 while TinyCore is using it (its `tce/` directory is there).
- TinyCore boots from an entry in the main GRUB menu (press Esc during boot if the menu is hidden).
- When the NVMe hangs, extension binaries (coreutils `tail`, `dmesg`, even the terminal) freeze too,
  because they are loop-mounted from the ESP. Use `busybox <cmd>` to keep working.
- Persist SSH across reboots: add `openssh.tcz` to `onboot.lst`, add `usr/local/etc/ssh` and
  `etc/shadow` to `/opt/.filetool.lst`, then run `filetool.sh -b`.
