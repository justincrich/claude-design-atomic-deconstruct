# REGEN-CASCADE

Upstream-variant propagation semantics. When a later phase surfaces a
`VARIANT_REQUEST` that targets a lower layer, the orchestrator pauses, updates
the lower layer, regenerates ALL of its artifacts, and resumes.

## THE CASCADE RULE

> When any phase N emits `VARIANT_REQUEST: {layer}=<name> variant=<class> recipe=<rules>`:
> 1. **Pause** the current dispatch in phase N.
> 2. **Dispatch targeted subagent** for the named component at its lower layer
>    with the variant spec (see PHASE-CONTRACTS.md § Orchestrator Regen Handler).
> 3. **Regenerate** that component's full artifact quartet:
>    - `{layer}/{name}/README.md` (new variant row in tables)
>    - `{layer}/{name}/{name}.html` (new variant section in preview)
>    - `{layer}/{name}/{name}.pdf` (re-render via docs/RENDER-ARTIFACTS.md)
>    - `{layer}/{name}/{name}.png` (re-render via docs/RENDER-ARTIFACTS.md)
> 4. **Update bundles** if the target is included in shared CSS:
>    - atom variant added → `molecules/_atoms.css`
>    - molecule variant added → `organisms/_molecules.css`
>    - organism variant added → `views/_organisms.css`
> 5. **Resume** the paused phase.

## DEPTH CAP

Maximum cascade depth per `VARIANT_REQUEST`: **3**.

- Views → atoms is depth 3 (view → organism → molecule → atom). Allowed.
- A view requesting an atom variant that requires a new token is depth 4
  (view → organism → molecule → atom → token). **Rejected** — rewrite as two
  separate escapes: `TOKEN_GAP` first (depth 0), then `VARIANT_REQUEST`.

If cascade attempts exceed depth 3, orchestrator logs to manifest and warns.

## CYCLE DETECTION

Keep a `visited` set per originating request, scoped `{layer}/{name}/{variant}`.

If during cascade the same tuple re-appears, abort with:
```
CASCADE_CYCLE detected: {a} → {b} → {a}
Original request: {source}
```
Surface to user; do not auto-resolve.

## IDEMPOTENCE

A `VARIANT_REQUEST` for a variant that already exists:
- Orchestrator notes the duplicate in manifest's `cascade_trail`.
- No re-write of the atom.
- Resume the paused phase immediately.

## EDITED-ARTIFACT PROTECTION

If a lower-layer artifact was edited by hand between phases (detected via
mtime mismatch with manifest's last known hash):
- Orchestrator warns the user before overwriting.
- Offer options:
  1. **Overwrite** (discard manual edits; lose them)
  2. **Merge** (re-dispatch subagent with the current edited file + variant spec; ask the subagent to preserve hand-edits where possible)
  3. **Skip** (leave the atom unchanged; the cascade REQUEST fails up to the caller)

## CASCADE AUDIT TRAIL

Every cascade emits an entry in `manifest.json`:

```json
{
  "cascade_trail": [
    {
      "timestamp": "2026-04-23T14:20:00-07:00",
      "source_phase": "organisms",
      "source_component": "nav",
      "target_layer": "atom",
      "target_component": "button",
      "variant_added": "atom-button--subtle-inverse",
      "depth": 2,
      "resolved": true,
      "regenerated": [
        "atoms/button/README.md",
        "atoms/button/button.html",
        "atoms/button/button.pdf",
        "atoms/button/button.png",
        "molecules/_atoms.css"
      ]
    }
  ]
}
```

## WHAT TRIGGERS A CASCADE VS. A TOKEN_GAP

| Request | Handler |
|---|---|
| "I need a color that doesn't exist" | `TOKEN_GAP` — add to tokens.css |
| "I need a font-size I don't have" | `TOKEN_GAP` — add to tokens.css |
| "I need a button variant that doesn't exist" | `VARIANT_REQUEST` — cascade to atom |
| "I need a molecule variant" | `VARIANT_REQUEST` — cascade to molecule |
| "I need a spacing step between existing ones" | `TOKEN_GAP` — add to tokens.css |
| "I need a new layout pattern that combines atoms I haven't seen yet" | This is a NEW molecule — don't cascade; add to this phase's molecule list if in Phase 3, or raise to user if in Phase 4+ |

## WHEN TO ESCALATE TO USER

- Cycle detected (see above)
- Depth > 3 required
- Edited artifact + user chose "skip"
- `VARIANT_REQUEST` names a component that doesn't exist at the target layer
  (user may want to add a new component instead)

The orchestrator pauses, surfaces the situation with full context, and asks
for direction.
