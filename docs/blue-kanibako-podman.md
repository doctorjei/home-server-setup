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
- Container creation takes ~74 s, mostly from kanibako's own permission adjustments at setup.

## After any unclean shutdown (before kanibako starts containers)
1. Check the LXC's underlying filesystem on blue (`pct config 300 | grep rootfs`).
2. `pct exec 300 -- su - agent -c "podman system check"` (report), then `--repair`
   (`--force` also removes containers on damaged layers).
3. `pct exec 300 -- su - agent -c "podman container cleanup --all"`
4. Anything created or changed shortly before the crash is suspect: recreate it.

## Storage layout (found 2026-10-05)
- LXCs on blue are managed by **kento** (OCI images run as PVE LXCs): `kanibako`
  (`ghcr.io/doctorjei/kanibako-lxc`, CT 300) and `jpn-vpn` (`ghcr.io/doctorjei/droste-thread`).
- CT 300's root filesystem is an **overlay built from an image in blue's host rootful Podman
  storage** (`/var/lib/containers/storage/overlay/<id>/diff` is the lower layer).
  ⚠️ **Never run `podman system reset`, `podman image prune` or `podman rmi` as root on blue's host
  without checking.** That can delete the root filesystem layers of kento LXCs (kanibako **and** jpn-vpn).
- Because `agent`'s Podman storage sat on that overlay (`backingFs=overlayfs`), native overlay was
  impossible and Podman used fuse-overlayfs (configured in `~/.config/containers/storage.conf`
  line 5, now commented out; backup at `storage.conf.bak-fuse`).
- **Fixed (2026-10-05):** btrfs subvolume `/var/lib/kanibako-podman` on blue (owned 1000:1000),
  bind-mounted into CT 300 via a raw LXC entry in `/etc/pve/lxc/300.conf`:
  `lxc.mount.entry: /var/lib/kanibako-podman home/agent/.local/share/containers none bind,create=dir 0 0`
  Verified: `backingFs=btrfs`, `useNativeDiff=true`, test container root is `overlay / overlay`.
  - Proxmox `mp0:` does **not** work here: PVE mounts it before kento's `lxc.hook.pre-mount`
    (`/var/lib/lxc/300/kento-hook`) builds the overlay root, which then covers it.
    `lxc.mount.entry` lines are applied after the hook (same mechanism as the existing
    `/net/workspace/kanibako/{projects,settings}` mounts).
  - If kento regenerates `300.conf`, this line must be carried into kento's config for the LXC.
  - Storage is disposable (images re-pullable). Agent data lives on `/net/workspace/kanibako/…`
    (network share) via the bind mounts above. Not covered by Proxmox backups.
- ⚠️ Lesson: `podman system reset` with `--root/--runroot` overrides still clears the user's shared
  runtime state (tmpdir, locks). It orphaned the running containers until an LXC restart +
  `podman system renumber`. Don't use it for test storage.

## To do
- [ ] Automate step 2 at boot, before kanibako starts (depends on how kanibako launches).
- [x] Switch `agent`'s storage to native overlay (done 2026-10-05, see above). Result after
      re-creating containers: `fuse-overlayfs` processes 0; blue memory used 12Gi → 5Gi,
      available 3.2Gi → 10Gi.
- [ ] Optional: share images host↔guest via a UID-1000-owned `additionalimagestores` store
      (needs matching `/etc/subuid`/`subgid` for `jjb` on blue and `agent` in CT 300).
- [ ] zram swap on blue.
- [ ] Optional: second 16GB DDR4 SODIMM if a physical second slot exists.
- [ ] Later: reconsider the privileged LXC (AI agent containers; escape = root on a cluster node).
- Noise: AppArmor `pasta` profile in complain mode logs `ALLOWED file_mprotect`; harmless.
