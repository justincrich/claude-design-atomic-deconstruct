# design-deconstruct

Reverse-engineer a concept HTML file into a token-governed atomic design system with full artifact quartets (README + HTML + PDF + PNG) at every layer.

## What it does

Takes a single "concept" HTML file and decomposes it through five sequential phases into a reusable design system:

1. **Tokens** — Semantic CSS custom properties, theme JSON files (light + dark), typography modules, and a primitives index page.
2. **Atoms** — Indivisible components (buttons, inputs, tags, icons). Each gets its own folder with a recipe README and a multi-variant preview page.
3. **Molecules** — Compositions of 2+ atoms (search bars, card headers, form fields). Compose by class; never redefine atom styling.
4. **Organisms** — Page-section units (nav, feed entries, footers). Compose from molecules + atoms.
5. **Views** — Full pages built from organisms, with responsive breakpoints for desktop and mobile.

Every artifact references theme tokens. Nothing hardcodes colors, spacing, or typography.

## Usage

```
/design-deconstruct <concept-html-path>
/design-deconstruct <concept-html-path> --output <dir>
/design-deconstruct <concept-html-path> --resume-from <phase>
/design-deconstruct <concept-html-path> --force
```

| Flag | Purpose |
|------|---------|
| `--output <dir>` | Override output directory (default: `./.spec/design/system`) |
| `--resume-from <phase>` | Resume from a specific phase (`tokens`, `atoms`, `molecules`, `organisms`, `views`) |
| `--force` | Full regeneration; ignores existing output |

## Output structure

```
.spec/design/system/
├── tokens/
│   ├── tokens.css            # light + dark custom properties
│   ├── theme.light.json
│   ├── theme.dark.json
│   └── theme.schema.json     # JSONSchema enforcing key parity
├── typography/
│   ├── fonts.css             # @font-face declarations
│   └── type-modules.css      # .type-h1, .type-body, etc.
├── atoms/
│   ├── _preview.css
│   ├── README.md             # atom index + composition matrix
│   └── {name}/
│       ├── README.md
│       ├── {name}.html
│       ├── {name}.pdf
│       └── {name}.png
├── molecules/
│   ├── _atoms.css            # shared atom bundle
│   ├── README.md
│   └── {name}/ ...
├── organisms/
│   ├── README.md
│   └── {name}/ ...
├── views/
│   ├── README.md
│   └── {name}/ ...
├── primitives/               # per-category PNG strips
├── tokens.html               # rendered primitives index
└── manifest.json             # run metadata + audit trail
```

## Quality bar

Every component artifact is verified against these rules at each phase boundary:

- **Zero hex literals** — all colors use `var(--{semantic})`
- **Zero raw typography values** — font-size, weight, line-height, and letter-spacing all use token variables
- **Zero raw px spacing** — padding, margin, and gap use `var(--space-*)`
- **No placeholder content** — previews render real markup from the concept
- **Every variant x state** rendered in both light and dark themes
- **Self-contained HTML** — links to `tokens.css` and `_preview.css`, no CDNs or build step
- **Composition purity** — molecules compose atoms by class, never redefine them

Failures trigger a re-dispatch of the offending component's subagent with the violating lines quoted.

## Regeneration cascade

If a later phase discovers a missing variant at a lower layer, it emits a `VARIANT_REQUEST`. The orchestrator:

1. Pauses the current phase
2. Dispatches a targeted subagent to extend the lower-layer component
3. Regenerates that component's full artifact quartet
4. Resumes the current phase

Cascades are hard-capped at 3 layers of recursion with cycle detection.

## Dependencies

- **`frontend-design:frontend-design`** — provides per-phase aesthetic briefing that steers each component's distinctive look
- **Headless Chrome** — required for PDF + PNG rendering (no fallback)

## When to use

| Use this skill | Don't use it for |
|---|---|
| Decomposing an existing concept HTML into a design system | Creating new designs from scratch (use `/frontend-design:frontend-design`) |
| Building a token-governed component library from a reference | Implementing components from specs (use `/pixel-perfect:build`) |

## Documentation

Detailed docs are loaded on demand during execution:

| File | Loaded when |
|------|------------|
| `docs/PHASE-CONTRACTS.md` | At subagent dispatch time |
| `docs/TOKEN-AUDIT.md` | At phase completion |
| `docs/REGEN-CASCADE.md` | When a `VARIANT_REQUEST` surfaces |
| `docs/RENDER-ARTIFACTS.md` | At component render time |
| `docs/SEMANTIC-TOKENS.md` | By Phase 1 subagent |
| `docs/OUTPUT-SCHEMA.md` | By Phase 1 subagent |
# claude-design-atomic-deconstruct
