# The Iteration & Sync Loop

proof-of-done is **long-term**: it improves every time an agent hits a new
"reported-done-but-wrong" trap. This file defines how that improvement stays
**consistent across every session and every project** — while still letting each
project specialize to its own stack. Both at once. That is the whole point.

---

## Two layers

- **Shared base (this repo)** — the generic, stack-agnostic source of truth:
  [PROTOCOL.md](PROTOCOL.md) (8 phases), [IRON-LAWS.md](IRON-LAWS.md) (R1…),
  [templates/](templates/). One source of truth, versioned by [`VERSION`](VERSION).
- **Project overlay** — each project's `docs/proof-of-done.md` (or
  `test-standard.md` / `LONG-TERM-TEST.md`): the stack-mapping table
  ([ADOPT.md](ADOPT.md) Step 2) + the project's own laws + a line pinning the
  base `VERSION` it was last synced to.

> Consistency comes from the shared base. Best-fit-per-project comes from the
> overlay. The loop below keeps the two in sync instead of drifting into N forks
> that each re-learn the same lessons.

---

## The loop — run it around every long-term test

### Before the run — pull the latest base
1. `git -C <base clone> fetch origin && git -C <base clone> log -1 origin/main`
   (or just read [`VERSION`](VERSION)).
2. If the base moved past your overlay's pinned `VERSION`:
   - read the new / changed laws,
   - update your overlay's pin to the new `VERSION`,
   - apply anything relevant to *this* run.

This is what guarantees **every session uses the same updated methodology** — the
base is the single upstream; each session re-pulls before it runs.

### After the run — push each learning to the right layer
For every new "done-but-wrong" trap you caught this run, **classify it**:

| Kind | Goes to | Effect |
|---|---|---|
| **Generic** — would bite *any* stack/project (a framework default, a race, an open redirect, a verification gap) | append a new `R{n}` to **this repo's** IRON-LAWS, bump `VERSION`, commit + push | every project gets it on its next pull |
| **Project-specific** — this stack's deploy quirk, this ERP's field semantics, this app's data shape | the **project overlay only** | stays where it's relevant |

**Litmus test:** *"Would a team on a completely different stack benefit from this?"*
Yes → base. No → overlay. When in doubt, write the generic principle to the base
and the concrete instance to the overlay.

---

## Why classify instead of "append everywhere"

- If a **generic** law lives only in one project's copy, every other project and
  session stays blind to it and re-faceplants the same way. Pushing it to the base
  is what makes the lesson *shared*.
- If a **project-specific** quirk gets pushed to the base, the base bloats with
  things 90% of adopters can't use, and the signal-to-noise of the iron laws drops.

Classification is the mechanism that makes **"all sessions, same updated
methodology"** and **"each project's test is best-fit for itself"** simultaneously
true — which is exactly the goal.

---

## Versioning

`VERSION` is date-based: `YYYY.MM.DD`, append `.N` for a second bump the same day
(`2026.06.15.2`). It is **monotonic**, so any session can compare its overlay's pin
against the base and immediately know whether it is stale. Bump it whenever you
add or change a law or a phase, in the same commit.

---

## One line

> One upstream base, many project overlays. Pull the base before you test; after
> you test, send generic scars upstream and project scars to the overlay. That is
> how the methodology stays one thing everywhere and the best thing anywhere.
