# UPS Power Budget

Sizing notes for the rack UPS. All figures are **wall draw** (power-supply conversion losses
included). Decision (2026-09-30): **CyberPower 1000W (1500VA) rack unit, ~$240**. It must be a
pure-sine "PFC" model, because mantis has an active-PFC PSU.

## Loads
| Device | Typical | Realistic max | Hard ceiling |
|--|--|--|--|
| mantis | ~60–100W | ~450W | ~650W (PSU rating; GPU capped at 200W, so unreachable) |
| loaf | ~10–15W idle, 150W inference | ~180W | ~255W (230W adapter) |
| 3× M73 Tiny | ~50–75W | ~150W | ~220W (3× 65W adapters) |
| HTPC (blue) | ~10–15W | ~60W | ~60W (55W device rating) |
| Network: Google Wifi + Tenda 16-port (12W max) + 4×2.5G/2×10G SFP+ switch | ~20–25W | ~35–40W | ~43–49W |
| Cable modem (Spectrum DOCSIS 3.1) | ~10–15W | ~20W | ~25–35W (adapter) |
| **Total** | **~160–390W** | **~895–900W** | **~1,260–1,270W** (theoretical) |

Observed regular load: **200–450W**.

## Assessment
- Regular load uses 20–45% of the rating. The realistic max (~900W, everything maxed at once) is
  still within the 1000W rating.
- The hard-ceiling sum is not reachable (mantis's GPU is capped at 200W, far below its 650W PSU).
- Everything maxed at once is rare (rough estimate ≈0.05% of the time). Overlap with an outage is
  rarer still.
- Runtime (CyberPower spec for CP1500PFCRM2U): ~3 min at full load, ~10 min at half load. At typical
  load, expect roughly 10–20 min.
- Overload (>1000W): on wall power, brief overloads are generally tolerated (alarm; breaker/shutdown
  if sustained). On battery, the output cuts off within seconds.
- The USB-C power distributor adds only its own conversion loss (already in the ÷0.9), not extra load.

## Plan
- Constraint: everything stays on the UPS.
- NUT: shut down **mantis and loaf** 1–2 min into an outage (largest and least critical loads). The
  network, M73s, and HTPC ride it out. Alert on `ups.load` above ~80%.
- After an outage: stagger power-on (BIOS "restore on AC loss" off or delayed for mantis and loaf).
- Data: once installed, log `ups.load` / `ups.realpower` with NUT `upslog` to replace estimates with
  measured peaks. Optional interim: `/usr/local/bin/powerlog.sh` on mantis (GPU + CPU watts every 5 s).
