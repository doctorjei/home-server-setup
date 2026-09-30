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

**Conclusion:** the drive and all filesystems are healthy. An earlier `ls` hang in the journal
directory did not reproduce and was not caused by on-disk damage (possible firmware/controller
hiccup). The boot failure is still under investigation. The journal was last written at 14:11 on
2026-09-30, so at least one recent boot got as far as systemd.

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
