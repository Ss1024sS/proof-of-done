# Long-term test report template

> Produced at sign-off (Phase 7). **Mandatory three tiers, never merged.** Every ✅ must carry independently re-checkable evidence.

---

## Long-term test report — <change topic>

**Change**: <one line on what changed>
**Environment**: <prod / staging + entry point>
**Commit**: <hash(es)>

### ✅ Verified (N)

> Each has objective, independently re-checkable evidence.

- **<feature>**: <evidence> — e.g. DB `SELECT` returned net shortage 4000 = SO-open 47072 − stock 43072, matches hand calc
- **<feature>**: <evidence> — e.g. endpoint returned correct 3-way distribution, schema fields complete
- …

### ⏳ Awaiting confirmation (M)

> Subjective judgment or external delivery — only promotable to ✅ after the receiver / reviewer confirms.

- **<feature>**: awaiting <whom> to confirm <what>
  - e.g. email went out (backend log confirms the callback fired), awaiting <recipient> to confirm inbox (incl. spam)
  - e.g. frontend filter deployed + fields verified, awaiting user hard-refresh to eyeball the render
- …

### ❌ Failed / untested (K)

> Failures / exceptions / uncovered scenarios — stated plainly, not hidden.

- **<feature>**: <reason / which scenario wasn't run / what error>
- …

### Phase 3 same-pattern sweep record

> The change's pattern + the verdict from grepping all occurrences (fix / no-fix + why).

- pattern: <what pattern changed>
- sweep result: <found N siblings, fixed X, Y left alone because…>

### Change manifest

- Files: <list>
- Deploy target: <env>
- Known leftovers / next iteration: <list>
