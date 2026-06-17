# The Wall — a wall-mounted Home Assistant family dashboard

This repo isn't a product or an add-on you can install. It's the **set of working documents** behind a custom 43″ touchscreen Home Assistant dashboard I built — and, more usefully, it's a record of **the method** I used to build it with an AI assistant doing the actual coding.

I'm not a software engineer or a Home Assistant expert. I know what I want it to look like and how it should feel, and I test everything on the real screen. The AI writes the code. What made that work — instead of devolving into a thousand disconnected chats — is the system of specs and rules in this repo. If you're trying to do the same thing, steal any of it.

---

## What's in here

Seven files, each with one job. The discipline that holds the whole thing together is simple: **any given topic lives in exactly one file. The others point to it, they never repeat it.** No duplication means nothing ever drifts out of sync.

| File | What it owns |
|---|---|
| **custom-instructions.md** | The assistant's standing operating manual — its role, its priorities, and the rules of how we work. Loaded at the top of the relationship, not re-explained each chat. |
| **family-dashboard-spec.md** | The physical build and the process: hardware, network, the kiosk/screen-off stack, burn-in defense, scope, roadmap, the master file manifest, and cost. |
| **gui-spec.md** | The design: look, interaction, motion, theme, and the tile library treated as a visual object. |
| **engine-spec.md** | The custom dashboard card's implementation: the animation/layout mechanics, the architecture, and the open engineering punch list. |
| **tiles-spec.md** | Per-tile wiring: which integration and entity sit behind each tile, the data-source specifics and gotchas, and what's actually built. |
| **sticky-notes-spec.md** | One feature big enough to earn its own canonical spec (a transparent note overlay on top of the dashboard). |
| **hardening-spec.md** | The robustness checklist — the "prove it survives days of real use" work that can only be done once the hardware is mounted. |

The four core specs are **family-dashboard / gui / engine / tiles** — think of them as *project · design · implementation · wiring*. The other two are a feature spec and a checklist that hang off the same system.

### These are living snapshots, not logs

The specs describe **the current state of the build and the plan forward — nothing else**. No change-logs, no "we switched from X to Y," no session-by-session history. When something changes, the old version gets deleted and the new truth is written in its place. The one deliberate exception is a single **Tile Build Status table** (in the family-dashboard spec) that tracks where each tile stands — it's the only thing kept as a running tracker.

This matters because the specs *are the AI's long-term memory.* A chat's context disappears when the chat ends. The specs don't. Keeping them clean and current is the entire reason I can open a brand-new chat and pick up exactly where I left off.

---

## How a session works

### Starting a chat

Every session opens the same way:

1. The assistant reads all the specs from the canonical source.
2. It renders the Tile Build Status table so we both see where everything stands.
3. It asks one question: **are we working on specs, or coding, this session?**
4. It waits for my direction before doing anything.

That ritual front-loads the shared context. By the time I say what I want, the assistant already knows the whole current state of the build.

### The build loop

Once we're coding, every change runs the same loop:

1. **I say what to build.**
2. **The assistant audits the real, current file** — it reads what's actually on disk right now, never reconstructs it from the spec. (The spec describes *intent*; only the live file is the literal truth. This rule prevents a huge class of "the AI rewrote something from memory and broke it" failures.)
3. **It builds, validates, and writes the file** — completely and atomically. No half-applied changes ever left on the target.
4. **I test it on the actual screen.** A hard-refresh on the real glass is the only proof that counts. Looks-right-in-theory doesn't.
5. **On my "it works,"** the change gets committed and pushed to version control.

No asking permission at every step. Once the plan for a change is agreed, the assistant makes the calls and executes the whole thing in one pass. The only things that go one-at-a-time are the steps *I* physically perform (installing something, clicking through a setup, wiring hardware).

### Ending a chat ("close-out")

Closing out means two things:

- **Commit and push** the working code to the repo.
- **Fold the session's decisions back into the specs** — update the right section, mark finished build steps done, refresh the file manifest and the Build Status table, and **delete anything that's no longer true.**

Spec edits happen *only* at close-out, never mid-build. That keeps the specs a stable reference during a working session instead of a moving target.

---

## Why it's built this way

A few principles do most of the heavy lifting:

- **One source of truth.** The live files on disk are canonical. The specs describe them; version control backs them up. When two sources disagree, the live file wins.
- **Single-source everything reusable.** A colour, a shared value, a reusable UI piece — defined once, referenced everywhere, never copy-pasted into a second definition.
- **Audit before building.** Read what's there and understand what depends on it before you touch it.
- **The real screen is the judge.** Not a linter, not the AI's confidence — the hard-refresh on the panel.
- **Build for yourself first.** This is a family dashboard, not a shipped product. The simplest path that works for *my* house beats a flexible, over-engineered one for hypothetical strangers.

The division of labour is the whole point: I bring product taste and real-world testing, the assistant brings execution and engineering rigor, and the specs are the shared brain that lets a non-coder direct a build like this without it falling apart.

---

## If you want to adapt this for your own build

The **method** is the reusable part — the silo'd specs, the session ritual, the build loop, the close-out discipline. The **specifics** (my cameras, calendars, thermostat, hardware, entity IDs) are tuned to my house and won't match yours.

Anywhere you see a marker like `>>>[SOMETHING]<<<`, that's a value I stripped before making this public — an IP address, a device ID, an account name, a location. Swap in your own. Everything I scrubbed was personal/identifying; none of it is part of the design or the method.

Start with **custom-instructions.md** to understand how the assistant is set up, then read the four core specs in order: family-dashboard → gui → engine → tiles. Happy to answer questions.
