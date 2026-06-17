# The Wall — Hardening Spec

How the Wall survives running untouched for weeks on real hardware: code resilience against Home Assistant churn, failure-state display rules, the diagnostics log, and the power/hardware hardening for a frequent-outage house. This is the standalone home for all robustness work — the four canonical specs point here.

## Maintenance rules (read first)

- This file owns hardening/resilience only: the code-survival patterns (error boundary, extractor firewall, dispatch wrapper, leak discipline), the two-stage failure-state rule, the health/diagnostics log pipe, staleness handling, reconnect behavior, and the power/hardware soak-test hardening.
- It does NOT own: the engine mechanics those patterns wrap (FLIP/packer/dispatch seam → engine-spec.md), per-tile data wiring (→ tiles-spec.md), the visual look of message/loading states (→ gui-spec.md), or the base hardware/kiosk/burn-in stack (→ family-dashboard-spec.md). Reference those by name; never restate them.
- A topic spanning files lives in exactly one. This file is where hardening lives; the canonical four carry one-line pointers back here.
- This is a live snapshot, not a log. No session history, no change-log. Describe what's decided and what's built now. Regenerate at close-out: fold new decisions in, mark built items, delete anything no longer true.
- Tiers are priority, not schedule. Tier 1 is the core that decides "runs for months" vs "needs a weekly reboot." If only Tier 1 + the Tier-2 log ever ship, the Wall is genuinely hardened. Tiers 3–4 are the long tail — real, but not blockers.

## Purpose — treat HA as a volatile dependency

Home Assistant is a self-updating backend that ships monthly and deprecates frontend APIs without consulting this codebase. The Wall embeds deeply (custom Web Component, `loadCardHelpers`, viewport measurement, the `hass` object contract), so an upstream change doesn't just look wrong — it can hard-crash the render loop. Add the physical reality: this panel runs **untouched for weeks**, never reloaded, in a house that **loses power often**. The bar is survive-unattended. Every pattern here exists so a single broken thing degrades gracefully into a clear message instead of taking down the Wall.

Core principle, applied everywhere: **when a tile can't show real data, say so plainly — never show stale data dressed up as current.** A frozen feed that looks live is worse than an honest "unavailable."

## Tier 1 — Core (build now)

The four that buy most of the value. All engine-spec territory (the patterns wrap existing engine seams).

1. **Per-face error boundary.** One throwing face must never blank the whole Wall. The dispatch seam wraps each face's render/extract; a throw is caught, that tile drops to its loading/unavailable state, the other eleven keep running. The single most important item here — it's the difference between "one tile looks off" and "the living-room screen goes black." **BUILT** — every render funnels through `_safeRender` (called by both `_renderFace` and `_renderBoardAdmin`) and every extractor through a try/catch in `_refreshLiveTiles`; a throw drops that one face to an honest "Unavailable" face (the `.t-*` glance grid) while the other tiles keep running. A thrown face is broken-now, not loading, so the boundary renders the unavailable state directly — the two-stage loading→unavailable escalation is item 8. Code on the share + `node --check`/anchor/brace validated; the runtime catch itself is pending an on-glass injected-fault test.

2. **Extractor firewall (required pattern, every `extract()`).** The extractor is the strict edge of the system — no raw `hass` data leaks past it into render logic. Every read uses optional chaining + nullish fallback (`state?.attributes?.x ?? null`) and explicitly traps the `"unavailable"`/`"unknown"` strings, emitting a clean `{ offline: true }` flag rather than letting `parseFloat` produce `NaN`. A bad number doesn't just show wrong text — it poisons the FLIP counter-scaling math. This is a *rule* for all extractors, not per-tile discretion. (Climate's `binary_sensor.climate_running` already models the availability trap — generalize it.)

3. **`_dispatch` try/catch wrapper.** The engine's single fire-and-forget write chokepoint (`tile._dispatch` → `hass.callService`) wraps its call in try/catch. A renamed/deprecated service fails quietly and writes a tagged entry to the diagnostics log (Tier 2) instead of throwing an unhandled rejection that stalls the interaction layer. **BUILT** — `tile._dispatch` wraps the `callService` in try/catch AND catches the returned promise’s rejection (the usual async failure mode); a renamed/failed service writes a `dispatch-error` breadcrumb instead of surfacing an unhandled rejection.

4. **Memory-leak discipline (required check).** Anything that sets a timer, event listener, or rAF loop MUST clean itself up when its tile unmounts. The radar loop already does this (self-cleaning rAF, stops on `!el.isConnected`, never `setInterval`) — make it a hard rule everywhere, audited at build. The other existential item: on a never-reloading screen, a small leak compounds over days into a crawl or a 3am crash.

## Tier 2 — Diagnostics pipe + failure states (build alongside)

5. **The health/diagnostics log (one pipe, two readers).** A single tagged-event stream is the spine of all diagnostics. Event types: face errored · service/dispatch failed · data stale · file/module 404 · entity offline/recovered · integration gone/re-auth · backup done/failed · resource breach · restart. Two views read it: **Quick Settings log** (recent, glanceable) and the **Control Room** (the full list). **The store is now BUILT** (the Settings maintenance-log pass): `www/diagnostics/diag.json` + the `diag_append.py` appender + the schema `{ts, category, type, source, message, color}`, with the first emitters live (`category:system` restarts + Core/OS/Supervisor updates) and two readers wired (Quick Settings + the System-tab maintenance log). The FIRST engine emitters are now live: the error boundary and `_dispatch` failures write `category:diagnostic` rows (`face-error`/`extract-error`/`dispatch-error`) through `_emitDiag` — self-guarded so it can never recurse, and deduped per `(type:source)` on a 5-min window so a repeating fault can’t hammer the disk. The System-tab reader was widened to surface `category:diagnostic` rows alongside `category:system`. What remains is the staleness-check emitter (item 9) into the SAME pipe; the store, schema, appender, and readers do not otherwise change. (Emitters validated; the runtime catch that feeds them is pending the on-glass injected-fault test.)

6. **Quick Settings log redesign.** Today's `homeQuick` notification ScrollList is inundated with surveillance events that already live on the Cameras tile. Change: **exclude camera/person/vehicle/doorbell events**, keep ALL other household notifications, and **add the system-health events** from the pipe above. Retention = a **recent window** (confirm the current live value on the box — likely 24h; the Control Room holds the full history, so Quick Settings can stay short).

7. **Structured breadcrumbs (not scattered prints).** The disciplined kind: the error boundary, dispatch failures, and staleness checks *are* the breadcrumbs — each writes a tagged entry (what broke · which tile · when) to the log pipe. No `console.log` sprinkling (invisible behind kiosk glass, useless). The payoff is on-glass diagnosis: tap Settings, see "Climate face errored 3 min ago" instead of guessing. This builds the project's diagnostics-first principle directly into the Wall.

8. **Two-stage failure states (no still frames, ever).** When a tile can't show live data it degrades in two stages, never to a frozen snapshot:
   - **Stage 1 — Loading (the blink):** a quiet LoadingFace for the brief gap (HA just restarted, data lands in ~seconds). Does not scream "broken" on every reboot.
   - **Stage 2 — Unavailable (clearly down):** once it's plainly not coming back (after a short escalation delay), an explicit message state in the Wall's design language — e.g. **"Camera unavailable"** — never a poster, never a last frame. A still feed masquerading as live is the worst outcome; an honest message means you always know what you're looking at.
   - Applies Wall-wide, not just cameras: any tile whose data is down shows the loading→message escalation, never a hopeful frozen value.

9. **Staleness vs. breakage.** A baked local file (radar/markets/media) or a `rest:` sensor can serve OLD data indefinitely with no error. Last-good degrade is correct (the cone cache already does it), but silently showing 3-day-old weather as current is its own failure. Tiles whose data carries a timestamp flag when the last-good copy is stale (a badge/note), so "last known good" never reads as "now." (Look = gui-spec.)

10. **Reconnect-storm handling.** HA restarts on every update, and the box reboots on every outage (frequent here). Define "connection dropped → wait → recover cleanly": don't flood HA with catch-up requests, don't freeze waiting on data mid-restart, don't re-fire stale event automations on boot. The software half of the power-resilience item in Tier 3.

## Tier 3 — Phase C / hardware (validate on the real panel + box)

Half of this is family-dashboard-spec, not code. None of it proves out until the soak test runs for days.

11. **Power resilience (frequent-outage house).** A whole-home surge protector stops spikes, not outages — the wrong tool for the actual problem. Each hard cut risks the NVMe if power dies mid-write; HA usually self-repairs but "usually" × "often" eventually loses.
    - **BIOS "restore on AC power loss" MUST be enabled** — box auto-boots after an outage, no walking to the enclosure to press power. Confirm before the wall is closed.
    - **Reconsider the no-UPS call.** The original "no UPS for v1" was decided before "loses power a lot" was known. At minimum a small UPS HA can talk to for **graceful shutdown** (the thing that actually protects the SSD), sited OUTSIDE the enclosure (battery = heat + bulk + wear). Reopens the family-dashboard Open Question.

12. **Box location — sink-cabinet vs in-wall enclosure (decide before the electrician).** A half-bath sits directly behind the panel wall; the vanity cabinet is a candidate home. Cabinet upside: open-air cooling (the i5-1235U boosts to 55W — a vented in-wall box is a heat trap), room for the UPS, trivial serviceability (matters in a reboot-prone house). Cabinet caveats: **bathroom humidity + leak risk** → box MUST be elevated/standoffed so a leak pools under it, not into it; **cable run** → HDMI + USB to the panel must stay within the USB ~16 ft passive limit and a clean HDMI length — measure before committing; minor fan noise in a quiet room. New family-dashboard Open Question; it moves where the electrician drops power + data.

13. **Scheduled restart — REJECTED (logged so it isn't re-litigated).** A nightly HA/box reboot is a Windows-era instinct that's wrong here: (a) a restart is a write-heavy event — it manufactures the exact mid-write SSD stress the outage hardening fights; (b) it MASKS slow leaks the soak test exists to catch, making the soak meaningless; (c) it blanks the living-room screen on a timer for no visible reason. **Replacement: a daily health-check** — HA inspects itself (memory, disk free, DB size, error count from the log pipe) and reports problems to the log; it watches and reports, it does not blindly reboot. (Distinct from the scheduled **screen-OFF** for burn-in — that's the display sleeping, already in the Phase-C plan, and stays.)

14. **Serve Lato + MDI locally on the box.** Promote the existing "drop the cloud font dependency" to-do to hardening: a CDN blip otherwise degrades every tile's text/icons at once.

15. **Disk-fill watch.** The daily health-check watches free space; HA's DB + logs grow (faster on a reboot-prone box) and a full drive corrupts silently.

16. **Kiosk watchdog.** Separate from HA — the kiosk browser (HAOSKiosk) leaks/locks over weeks too. If it goes unresponsive, something reloads it rather than leaving a frozen screen. Known wall-dashboard failure mode; folds into the Phase-C soak.

17. **Module 404 resilience.** Every `engine/` and `faces/` file is its own 404 surface; one missing file kills the import chain and can blank the Wall. Filenames stay case-exact, imports verified; a failed module load logs to the pipe. The on-glass hard-refresh stays the real proof.

18. **Don't auto-update HACS cards** (especially `custom:webrtc-camera`) — same discipline as not auto-updating HAOS. Pin versions; test an update on the VM staging copy before the live box.

19. **Confirm backup restore + key survival.** Automatic HA Backups + GitHub are the floor. The migration already proved a real restore works (better than most). Confirm the **backup encryption key** still lives somewhere that survives the box dying — without it the backups are bricks. (Already flagged load-bearing in family-dashboard; this is a recurring re-confirm.)

## Tier 4 — Logged, low urgency

- **Empty vs. broken.** Zero calendar events today is a valid quiet state ("nothing today"), not an error look. Don't conflate empty-but-fine with something-broke.
- **First-paint race.** On cold boot, tiles render before `hass` has handed over all data. Every tile shows its LoadingFace for that gap, not a flash of garbage.
- **Data-push ceiling.** HA pushes every entity change; a flapping entity could trigger draws faster than the frame budget allows. A "ignore updates faster than I can draw" ceiling protects the budget. (Per-face repaint discipline already helps.)
- **Timer/automation pileup after restart.** On HA boot, delay/timer automations shouldn't re-fire or spam the log (garage-open timer, re-notifications).
- **Touchscreen recalibration drift.** PCAP taps can land off over time. Maintenance item, Phase C.
- **DST + sun-dial math.** The celestial dial and any sunrise/sunset math survive the clock jumping an hour twice a year and day-boundary rollovers.
- **Runtime is fully local.** The Wall reads its face files from the box's own storage, never across a network path at runtime (the Samba mount is dev-only). An explicit line so a network-read never creeps in.

## Out of scope for v1 (don't over-build)

- Multi-user / concurrent edits — one screen, one wall.
- Security / malicious-input hardening — family-only, local network.
- Offline-first / works-without-HA — if HA is down the house is down; last-good display is enough.
- NVMe physical death — restore from backup, accepted risk.

## Pointers (added to the canonical specs at close-out)

- **engine-spec.md** → Resilience patterns (per-face error boundary, extractor firewall, `_dispatch` wrapper, leak discipline, two-stage failure states, module 404) → hardening-spec.md.
- **tiles-spec.md** → Per-tile staleness + offline message states; the systems/maintenance log is BUILT as the unified diagnostics pipe (`diag.json` + `diag_append.py`); hardening adds the engine emitters → hardening-spec.md.
- **gui-spec.md** → The look of loading vs. unavailable message states + the staleness badge → hardening-spec.md.
- **family-dashboard-spec.md** → Power resilience (BIOS auto-on, UPS reversal), box location (sink cabinet vs enclosure), scheduled-restart rejection + daily health-check, local fonts, disk/kiosk watchdog, backup-restore confirm → hardening-spec.md.
