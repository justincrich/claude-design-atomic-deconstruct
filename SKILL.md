---
name: design-deconstruct
description: Take a single "concept" HTML file and reverse-engineer it into a complete, token-governed atomic design system through five sequentially-dependent TaskList phases — tokens → atoms → molecules → organisms → views. Every artifact (README + HTML preview + PDF + PNG) references theme tokens; nothing hardcodes. Lower-layer variants can be added upward propagation — if an organism needs a button variant that doesn't exist, the atom is extended and all its artifacts are regenerated before the organism phase resumes. Depends on frontend-design:frontend-design for per-phase aesthetic briefing. USE when the user provides a concept HTML and wants it turned into a reusable design system with full artifact quartets per component. NOT for creating new designs (use /frontend-design:frontend-design) or implementing from specs (use /pixel-perfect:build).
model: opus
---

# design-deconstruct

Sequentially decompose a concept HTML into a governed atomic design system. Five phases, one TaskList chain, zero hardcoded values.

## THE FIVE SEQUENTIAL PHASES

```
concept.html
     │
     ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ T1 TOKENS   │───►│ T2 ATOMS    │───►│ T3 MOLECULES│───►│ T4 ORGANISMS│───►│ T5 VIEWS    │
│             │    │             │    │             │    │             │    │             │
│ tokens.css  │    │ atoms/{N}/  │    │ molecules/  │    │ organisms/  │    │ views/      │
│ theme.*.json│    │  quartet    │    │  quartet    │    │  quartet    │    │  quartet    │
│ fonts.css   │    │             │    │             │    │             │    │             │
│ type-mods.css│   │ _preview.css│    │ _atoms.css  │    │             │    │             │
│ tokens.html │    │             │    │             │    │             │    │             │
│ primitives/ │    │             │    │             │    │             │    │             │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
      │                 ▲ TOKEN_GAP         ▲ TOKEN_GAP / VARIANT_REQUEST → cascades DOWN
      └─────────────────┴───────────────────┴────────────────────────────────────┘
```

**Tasks chain via `addBlockedBy`**: T2 blocked by T1, T3 by T2, T4 by T3, T5 by T4. No task starts until its predecessor completes.

**Cascade** (docs/REGEN-CASCADE.md): a later phase can surface a `VARIANT_REQUEST` for a missing lower-layer variant. The orchestrator pauses the current phase, extends the lower layer, regenerates that component's full quartet (README + HTML + PDF + PNG), then resumes. Hard-capped at 3-layer recursion.

## QUARTET CONTRACT (every component)

```
{layer}/{name}/
├── README.md         recipe — atoms-used table, token references per property
├── {name}.html       preview — every variant × state × both themes
├── {name}.pdf        paged print rendering
└── {name}.png        single-strip snapshot (1320px wide, for LLM reference)
```

This contract applies at the atom, molecule, organism, and view layers. The orchestrator renders `.pdf` + `.png` via docs/RENDER-ARTIFACTS.md after each subagent emits README + HTML.

## THE NON-NEGOTIABLE QUALITY BAR

Every component artifact MUST satisfy these (verified by docs/TOKEN-AUDIT.md grep rules at end of each phase):

1. **Zero hex literals** — every color is `var(--{semantic})`.
2. **Zero numeric font-size / font-weight / line-height / letter-spacing** — use `var(--font-size-*)`, `var(--font-weight-*)`, `var(--line-height-*)`, `var(--tracking-*)`.
3. **Zero raw px in padding / margin / gap** — use `var(--space-*)` scale.
4. **No placeholder content** — every preview renders real markup (titles, labels, actual glyphs) from the concept.
5. **Every variant × state rendered** in a grid using `.is-{name}` forced classes.
6. **Self-contained HTML** — `<link rel="stylesheet" href="…/tokens.css">`, `<link rel="stylesheet" href="…/_preview.css">`, no CDNs, no build.
7. **Composition purity** — molecules import atom classes, never redefine atom styling; organisms import molecules; views import organisms. Enforced by shared `_atoms.css` at the molecule layer.

If any artifact fails, the orchestrator re-dispatches that component's subagent with the offending lines quoted.

## QUICK REFERENCE

```
SYNTAX:
  /design-deconstruct <concept-html-path>
  /design-deconstruct <concept-html-path> --output <dir>
  /design-deconstruct <concept-html-path> --resume-from <tokens|atoms|molecules|organisms|views>
  /design-deconstruct <concept-html-path> --force

DEFAULT OUTPUT:
  ./.spec/design/system

PER-PHASE AGENT:
  Skill("frontend-design:frontend-design", "{phase}-layer brief for {component}")
  Agent(subagent_type="frontend-designer", prompt=docs/PHASE-CONTRACTS.md § phase)

ESCAPE PROTOCOLS (subagent → orchestrator):
  TOKEN_GAP:  --{key} := {value} ({reason})            # extend tokens.css
  VARIANT_REQUEST: atom|molecule|organism={name}
                  variant={class} recipe={css-rules}    # regen lower layer
```

## ORCHESTRATION ALGORITHM

```
[1] PARSE INPUT
    Required: <concept-html-path>   — single file, bundled OK (decoded in [3])
    Optional: --output, --resume-from, --force
    Verify concept file exists and is readable.

[1.5] DETECT EXISTING OUTPUT (re-run UX)
    Resolve output directory (default: ./.spec/design/system).
    IF --force flag set → skip detection, full regeneration.
    IF output directory exists AND contains component folders:
      Glob all {layer}/{name}/README.md files to build inventory.
      IF inventory is non-empty:
        PHASE_ORDER = [tokens, atoms, molecules, organisms, views]
        ASK 1 — Phase selection:
          AskUserQuestion with multiSelect=true:
            Question: "Found {N} existing components. Which phases to regenerate?"
            Options: each phase name (tokens, atoms, molecules, organisms, views)
            Button label: "Regenerate"
          Build "dirty set" from selected phases.
          IF nothing selected → warn "No phases selected for regeneration" → EXIT.

        CASCADE — Upward dependency propagation:
          For each selected phase at index I:
            Add all phases at index > I to dirty set (they depend on I).
          Example: user selects atoms → dirty set becomes {atoms, molecules, organisms, views}.
          Example: user selects tokens → dirty set becomes {all 5 phases}.
          Example: user selects views only → dirty set stays {views}.
          cascade_added = dirty set minus user's original selections.

        ASK 2 — Confirmation screen:
          AskUserQuestion (single select):
            Question: "Regeneration plan:"
              • {user-selected phases} (selected)
              • {cascade_added phases} (added because {lowest-selected-phase} changed)
            Options: "Confirm" / "Cancel"
          IF "Cancel" → EXIT without regenerating.
          IF "Confirm" → proceed with dirty set as the regeneration scope.

        Mark phases NOT in dirty set as "cached" — skip their subagent dispatch.
    IF no existing output → full run (normal first-run behavior).

[2] PLAN THE 5-TASK CHAIN
    TaskCreate one task per phase. Chain with addBlockedBy:
      T1 "deconstruct tokens"
      T2 "deconstruct atoms"            blockedBy T1
      T3 "deconstruct molecules"        blockedBy T2
      T4 "deconstruct organisms"        blockedBy T3
      T5 "deconstruct views"            blockedBy T4

[3] PREPARE OUTPUT + DECODE CONCEPT
    mkdir -p <output>/{tokens,typography,atoms,molecules,organisms,views,primitives}
    IF concept is a bundled template (has <script type="__bundler/template">):
      Decode via base64 + gzip manifest extraction (see docs/PHASE-CONTRACTS.md § 1
      for the Python snippet) so subagents can read the source components.
    Copy _preview.css skeleton to atoms/_preview.css.
    Write <output>/manifest.json with concept path + run timestamp.

[4] EXECUTE PHASE 1 — TOKENS  (T1)
    TaskUpdate T1 → in_progress.
    Skill("frontend-design:frontend-design", "tokens-layer brief")
    Dispatch ONE frontend-designer subagent, prompt = docs/PHASE-CONTRACTS.md § 1:
      INPUT:  concept HTML path (and decoded source if bundled)
      MUST produce:
        tokens/tokens.css              light + dark, identical keys
        tokens/theme.light.json        JSON mirror
        tokens/theme.dark.json         JSON mirror
        tokens/theme.schema.json       JSONSchema enforcing key parity
        typography/fonts.css           @font-face + font stack
        typography/type-modules.css    .type-h1/h2/title/body/meta/label/code
        tokens.html                    rendered primitives index (two-theme)
    After return:
      • Validate theme.light.json + theme.dark.json against schema (structural parity)
      • Render primitives PNGs per category (docs/RENDER-ARTIFACTS.md)
      • Run docs/TOKEN-AUDIT.md on tokens.html (MUST pass — it IS the tokens)
    TaskUpdate T1 → completed.

[5] EXECUTE PHASE 2 — ATOMS  (T2)
    TaskUpdate T2 → in_progress.
    Identify atom inventory from concept (single-purpose, not decomposable).
    FOR EACH atom (SEQUENTIAL — no fan-out):
      Skill("frontend-design:frontend-design", "atom-layer brief for {name}")
      Dispatch frontend-designer subagent, prompt = docs/PHASE-CONTRACTS.md § 2
        (substituting atom name + concept excerpt + tokens.css path):
          MUST produce:
            atoms/{name}/README.md     recipe with atoms-used=none, token table
            atoms/{name}/{name}.html   preview w/ every variant × state × theme
          MUST reference only theme tokens.
          ESCAPE: TOKEN_GAP: --{key} := {value} ({reason})
      ON TOKEN_GAP:
        • Orchestrator opens tokens/tokens.css
        • Appends the new semantic alias (additive only; no renames)
        • Re-renders tokens.html + primitives PNG
        • Continues to next atom
      Verify via docs/TOKEN-AUDIT.md grep pass on the atom's HTML.
        Any violation → re-dispatch with offending lines.
      Render PDF + PNG via docs/RENDER-ARTIFACTS.md.
    Update atoms/README.md with the full atom index + atoms-used matrix.
    TaskUpdate T2 → completed.

[6] EXECUTE PHASE 3 — MOLECULES  (T3)
    TaskUpdate T3 → in_progress.
    Identify molecules (2+ atom compositions) from concept.
    Build shared molecules/_atoms.css by bundling each atom's CSS rules
      (so molecule previews compose atoms by class, importing one stylesheet).
    FOR EACH molecule (SEQUENTIAL):
      Skill("frontend-design:frontend-design", "molecule-layer brief for {name}")
      Dispatch frontend-designer subagent, prompt = docs/PHASE-CONTRACTS.md § 3:
        INPUT:  molecule name + concept excerpt + atoms index + _atoms.css path
        MUST produce: molecules/{name}/ README + HTML
        CONSTRAINTS:
          • Compose atoms by class; NEVER redefine atom styling
          • README lists every atom used in a table
          • Any molecule-local value must resolve to a token
        ESCAPE:
          • TOKEN_GAP (same as Phase 2)
          • VARIANT_REQUEST: atom={name} variant={class} recipe={css-rules}
      ON VARIANT_REQUEST  (docs/REGEN-CASCADE.md):
        • Load atom at atoms/{name}/
        • Append the new variant to atoms/{name}/{name}.html + README.md
        • Regenerate atom PDF + PNG
        • Update molecules/_atoms.css with the new class
        • Resume current molecule dispatch
      Audit + render molecule quartet.
    Update molecules/README.md with the atoms-used matrix per molecule.
    TaskUpdate T3 → completed.

[7] EXECUTE PHASE 4 — ORGANISMS  (T4)
    TaskUpdate T4 → in_progress.
    Identify organisms (page-section units — nav, feed-entry, footer, project-row…).
    FOR EACH organism (SEQUENTIAL):
      Skill("frontend-design:frontend-design", "organism-layer brief for {name}")
      Dispatch frontend-designer subagent, prompt = docs/PHASE-CONTRACTS.md § 4:
        INPUT:  organism name + concept excerpt + atoms + molecules indexes
        MUST produce: organisms/{name}/ README + HTML
        CONSTRAINTS: compose from molecules + atoms; no redefinition
        ESCAPE: TOKEN_GAP | VARIANT_REQUEST (target atom OR molecule)
      Cascade per docs/REGEN-CASCADE.md (multi-layer updates allowed; max depth 3).
      Audit + render.
    Update organisms/README.md.
    TaskUpdate T4 → completed.

[8] EXECUTE PHASE 5 — VIEWS  (T5)
    TaskUpdate T5 → in_progress.
    Identify views (full pages — feed, projects, about, entry-detail).
    FOR EACH view (SEQUENTIAL):
      IF view is in cached set (from [1.5]) → skip, preserve existing artifacts.
      Skill("frontend-design:frontend-design", "view-layer brief for {name}")
      Dispatch frontend-designer subagent, prompt = docs/PHASE-CONTRACTS.md § 5:
        INPUT:  view name + concept excerpt + atoms + molecules + organisms
        MUST produce: views/{name}/ README + HTML
        CONSTRAINTS:
          • Compose from organisms (primarily) + lower layers; no redefinition
          • Views MUST use stacked vertical layout — each variant in its own
            full-width section. Do NOT use .two-up or side-by-side grids.
            Light and dark theme panes stack vertically as separate sections.
          • Include @media queries for responsive breakpoints:
            Desktop: default (max-width per layout token)
            Mobile: max-width 375px — single column, stacked nav, full-width entries
        ESCAPE: TOKEN_GAP | VARIANT_REQUEST (any layer)
      Cascade through any triggered layers.
      Audit + render (desktop PNG at 1320px + mobile PNG at 375px per
        docs/RENDER-ARTIFACTS.md § View Rendering).
    Update views/README.md.
    TaskUpdate T5 → completed.

[9] FINAL AUDIT
    Run docs/TOKEN-AUDIT.md across ALL layers:
      • zero hex literals
      • zero font-size/weight/line-height/tracking numeric literals
      • zero raw px spacing literals
    Any violation → re-dispatch offending component subagent with line numbers.
    Cap at 2 re-dispatches per component; log to <output>/manifest.json "gaps" array
      if still failing and surface in report.

[10] REPORT
    Print summary:
      tokens: N semantic keys, M typography modules, K primitives PNGs
      atoms: N folders × avg V variants
      molecules / organisms / views: counts + atoms-used coverage
    Emit paths:
      <output>/tokens.html                  primitives index
      <output>/{layer}/README.md            per-layer index
      <output>/manifest.json                run metadata + any gaps
```

## AGENT DISPATCH RULES

1. **Every subagent runs sequentially** within its phase — no parallel fan-out. The user chose strict sequentiality so cascades can be handled cleanly.
2. **Every subagent dispatch includes the aesthetic brief** from `Skill("frontend-design:frontend-design", …)`. The `frontend-design` skill's guidance steers the component's distinctive look; `design-deconstruct` owns the structural discipline.
3. **Every subagent prompt embeds the Non-Negotiable Quality Bar verbatim** and lists the exact escape protocol strings (`TOKEN_GAP:`, `VARIANT_REQUEST:`) it may emit.
4. **Subagents write directly to deterministic paths** — no scripts, no generators. `Read/Write/Edit/Glob/Grep` plus `frontend-design` consultation only.
5. **Orchestrator owns all rendering** (PDFs, PNGs) — subagents never shell out to Chrome. This keeps subagent tool surfaces minimal.
6. **Orchestrator owns tokens.css mutations** — subagents propose via `TOKEN_GAP`; the orchestrator appends.
7. **Cascade regen is orchestrator-driven** — on `VARIANT_REQUEST`, the orchestrator dispatches a *targeted* frontend-designer subagent limited to adding the new variant to the specified component (not a full re-author).

## REGENERATION CASCADE (summary)

When a `VARIANT_REQUEST` surfaces at layer N:
1. Orchestrator pauses the current phase (N's current component).
2. Dispatches a targeted frontend-designer subagent to layer N-1 (or N-2, N-3) with the variant spec.
3. Target subagent updates the lower component's README (new variant row) + HTML (new state cell).
4. Orchestrator re-renders the lower component's PDF + PNG.
5. If the lower component is in the shared `_atoms.css` / atom bundle, orchestrator updates that file.
6. Orchestrator resumes the current phase.

See **`docs/REGEN-CASCADE.md`** for:
- depth cap (3 layers max)
- cycle detection
- edited-artifact protection (mtime vs manifest hash)

## SUCCESS CRITERIA

1. Five TaskList tasks chain via `addBlockedBy` and complete in order.
2. Every component artifact quartet present (README + HTML + PDF + PNG).
3. Zero hex / typography numeric / px-spacing literals in any atom/molecule/organism/view CSS (docs/TOKEN-AUDIT.md passes).
4. `theme.light.json` and `theme.dark.json` validate identically against `theme.schema.json`.
5. `tokens.html` opens and renders every primitive in both themes.
6. Every layer's `README.md` contains an atoms-used / composition matrix.
7. All `VARIANT_REQUEST` cascades left no stale artifacts — every touched component's PDF + PNG was regenerated in the same phase.
8. Opening any `{layer}/{name}/{name}.html` in a browser shows real markup using `theme.css` (not placeholders).

## EXAMPLES

```bash
# Full deconstruction from a concept file
/design-deconstruct /Users/justinrich/Projects/experience-surface/.spec/design/concepts.html

# Resume after interruption (skill reads TaskList state)
/design-deconstruct concepts.html --resume-from organisms

# Force full re-run (clears TaskList, starts over)
/design-deconstruct concepts.html --force

# Alternate output dir
/design-deconstruct concepts.html --output ./.spec/design-system
```

## EDGE CASES

| Case | Handling |
|---|---|
| Concept HTML missing | Exit with message; suggest path |
| Concept is bundled (`__bundler/template`) | Decode via base64 + gzip extraction before Phase 1 (docs/PHASE-CONTRACTS.md § 1 has the Python snippet) |
| Phase 1 emits non-parity light/dark JSON | Fail schema validation; re-dispatch Phase 1 with diff |
| Atom contains raw hex / font-size literal | docs/TOKEN-AUDIT.md fails; re-dispatch atom subagent with offending lines |
| VARIANT_REQUEST for existing variant | Merge into existing; log in manifest audit trail |
| VARIANT_REQUEST cascades more than 3 layers | Hard-cap recursion; warn |
| Headless Chrome unavailable | Exit with install prompt; no fallback |
| `--resume-from X` | Mark X + successors pending in TaskList; re-execute from X |
| User edited a lower-layer artifact by hand between phases | mtime vs manifest hash mismatch; prompt before overwriting |
| Subagent emits `>preview<` or `TODO` placeholder | Reject + re-dispatch with "no placeholder" reminder |
| Two successive phases complete with no VARIANT_REQUEST | Normal — most chains don't cascade |

## PROGRESSIVE DISCLOSURE

**Core skill** (this file): orchestration + task chain + quality bar + success criteria.

**Load on demand**:
- `docs/PHASE-CONTRACTS.md` — full subagent dispatch prompts per phase (loaded at dispatch time)
- `docs/TOKEN-AUDIT.md` — grep patterns + reject-and-regen policy (loaded at phase completion)
- `docs/REGEN-CASCADE.md` — cascade semantics + recursion limits (loaded only when a VARIANT_REQUEST surfaces)
- `docs/RENDER-ARTIFACTS.md` — Chrome headless + PIL crop recipe (loaded at component render time)
- `docs/SEMANTIC-TOKENS.md` — token-naming vocabulary (loaded by Phase 1 subagent)
- `docs/OUTPUT-SCHEMA.md` — JSON schemas for theme files (loaded by Phase 1 subagent)

**Why this structure**:
- Orchestration algorithm used every run → in this file.
- Phase prompts loaded only at dispatch (big prompts).
- Audit rules loaded at phase completion only.
- Cascade rules loaded rarely (only when a variant request surfaces).
- Render recipe loaded once per component.
