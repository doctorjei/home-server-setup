# blue / kanibako: Podman Storage Notes

blue: Intel N95, 16GB DDR4 SODIMM (1 of 2 reported slots used; Intel's official max is 16GB, but
32GB usually works, so check the board physically). No swap. Proxmox node; the kanibako server is
privileged LXC **300**. Inside it, user `agent` (UID 1000, shown as `jjb` on the host) runs rootless
Podman 5.4.2 with **fuse-overlayfs**.

## Incident (2026-10-05)
- Memory pinned; two `fuse-overlayfs` processes at ~1GB each, 25%/15% CPU, `D` state.
- `podman rm kanibako-kanibako` hung in "Removing"; Podman logged
  `Found incomplete layer "bc05d960…", deleting it`, which was the writable layer of the
  **running** `kanibako-fodder` (disposable, recreated).
- No OOM kills, no hung-task warnings.
- **Root cause:** the UPS failure caused five boots on 10-03/04. During boot -3 (21:36–22:44 UTC),
  `kanibako-kanibako` and `kanibako-fodder` were removed and re-created; power was lost again at
  22:44. Their layer metadata ("incomplete" flag) was never cleared on disk. Both broke a day later.
- `podman system check --quick` after cleanup: clean.

## After any unclean shutdown (before kanibako starts containers)
1. Check the LXC's underlying filesystem on blue (`pct config 300 | grep rootfs`).
2. `pct exec 300 -- su - agent -c "podman system check"` (report), then `--repair`
   (`--force` also removes containers on damaged layers).
3. `pct exec 300 -- su - agent -c "podman container cleanup --all"`
4. Anything created or changed shortly before the crash is suspect: recreate it.

## To do
- [ ] Automate step 2 at boot, before kanibako starts (depends on how kanibako launches).
- [ ] Switch `agent`'s storage from fuse-overlayfs to native kernel overlay (LXC is privileged;
      confirm `nesting=1`, and if the rootfs is ZFS, need ZFS ≥ 2.2). May require
      `podman system reset`, so back up named volumes first.
- [ ] zram swap on blue.
- [ ] Optional: second 16GB DDR4 SODIMM if a physical second slot exists.
- [ ] Later: reconsider the privileged LXC (AI agent containers; escape = root on a cluster node).
- Noise: AppArmor `pasta` profile in complain mode logs `ALLOWED file_mprotect`; harmless.
