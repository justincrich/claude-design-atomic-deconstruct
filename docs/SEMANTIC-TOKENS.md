# Semantic Tokens — design-deconstruct v2

A semantic token sits between raw concept tokens (`color.n-0`, `space.sp-3`) and component specs. It names the **purpose** of a value (`surface.page`, `spacing.between`) so components reference intent, not values. Theme swaps, brand tweaks, and accessibility adjustments become changes to the **mapping**, not to every component that used the color.

This doc defines the canonical v2 vocabulary, the derivation algorithm, and the mapping schema.

## The two-layer model

```
┌──────────────────────┐
│  Concept tokens      │  color.n-0 = #f7f6f3 (light) / #151513 (dark)
│  (raw values)        │  color.p-400 = #8c3a28 (light) / #c85541 (dark)
│  foundations.json    │  space.sp-3 = 32px
└──────────────────────┘
           │
           │   aliasFor (per-theme)
           ▼
┌──────────────────────┐
│  Semantic tokens     │  semantic.surface.page      → color.n-0
│  (purpose-named)     │  semantic.accent.primary    → color.p-400
│  semantic-tokens.json│  semantic.spacing.between   → space.sp-3
└──────────────────────┘
           │
           │   tokenRef
           ▼
┌──────────────────────┐
│  Component specs     │  button.primary.bg = { tokenRef: "semantic.accent.primary" }
│  atoms.json etc.     │
└──────────────────────┘
           │
           │   emitted as
           ▼
┌──────────────────────┐
│  theme.css           │  :root        { --accent-primary: #8c3a28; }
│  (CSS custom props)  │  [data-theme="dark"] { --accent-primary: #c85541; }
└──────────────────────┘
```

Components read CSS vars (`var(--accent-primary)`); themes flip the mapping; nothing else changes.

## Canonical v2 vocabulary

Semantic IDs follow `semantic.{category}.{name}`. Names are kebab-case. Every token below is optional — a skill run only emits what the source actually uses.

### surface — backgrounds

| ID | CSS Var | Purpose |
|---|---|---|
| `semantic.surface.page` | `--surface-page` | Base page background |
| `semantic.surface.soft` | `--surface-soft` | Subtle layered surface (panel within page) |
| `semantic.surface.sunken` | `--surface-sunken` | Recessed region (code block, inset card) |
| `semantic.surface.raised` | `--surface-raised` | Elevated panel above page |
| `semantic.surface.overlay` | `--surface-overlay` | Modal / popover backdrop tint |
| `semantic.surface.inverse` | `--surface-inverse` | Inverted surface (dark panel on light app) |

### text — foreground content

| ID | CSS Var | Purpose |
|---|---|---|
| `semantic.text.heading` | `--text-heading` | Primary heading color |
| `semantic.text.body` | `--text-body` | Body text |
| `semantic.text.muted` | `--text-muted` | Secondary text |
| `semantic.text.faint` | `--text-faint` | Tertiary / caption |
| `semantic.text.inverse` | `--text-inverse` | Text on an accent/dark surface |
| `semantic.text.link` | `--text-link` | Default link |
| `semantic.text.link-hover` | `--text-link-hover` | Hovered link |
| `semantic.text.disabled` | `--text-disabled` | Disabled text |

### border — strokes & dividers

| ID | CSS Var | Purpose |
|---|---|---|
| `semantic.border.default` | `--border-default` | Default component border |
| `semantic.border.subtle` | `--border-subtle` | Hairline divider |
| `semantic.border.strong` | `--border-strong` | Emphasized border |
| `semantic.border.focus` | `--border-focus` | Focus ring color |
| `semantic.border.disabled` | `--border-disabled` | Disabled component border |

### accent — brand / primary

| ID | CSS Var | Purpose |
|---|---|---|
| `semantic.accent.primary` | `--accent-primary` | Primary brand color |
| `semantic.accent.primary-hover` | `--accent-primary-hover` | Hovered primary |
| `semantic.accent.primary-pressed` | `--accent-primary-pressed` | Active/pressed primary |
| `semantic.accent.primary-subtle` | `--accent-primary-subtle` | Tinted background (badge bg, selected row) |
| `semantic.accent.primary-fg` | `--accent-primary-fg` | Foreground color on primary bg |
| `semantic.accent.secondary` | `--accent-secondary` | Secondary accent (optional) |
| `semantic.accent.secondary-subtle` | `--accent-secondary-subtle` | Tinted secondary bg |

### state — feedback colors

| ID | CSS Var | Purpose |
|---|---|---|
| `semantic.state.success.fg` | `--state-success-fg` | Success foreground |
| `semantic.state.success.bg` | `--state-success-bg` | Success background |
| `semantic.state.success.dot` | `--state-success-dot` | Success indicator dot |
| `semantic.state.warning.fg/.bg/.dot` | `--state-warning-*` | Warning |
| `semantic.state.danger.fg/.bg/.dot` | `--state-danger-*` | Error / destructive |
| `semantic.state.info.fg/.bg/.dot` | `--state-info-*` | Informational |
| `semantic.state.neutral.dot` | `--state-neutral-dot` | Paused / archived indicator |

### spacing — scale

| ID | CSS Var | Purpose |
|---|---|---|
| `semantic.spacing.micro` | `--spacing-micro` | Icon gaps, tag padding |
| `semantic.spacing.internal` | `--spacing-internal` | Inside-component padding |
| `semantic.spacing.between` | `--spacing-between` | Between components |
| `semantic.spacing.section` | `--spacing-section` | Between page sections |

### typography — type roles

| ID | CSS Var | Purpose |
|---|---|---|
| `semantic.type.page-title` | `--type-page-title-size` | Page titles |
| `semantic.type.section-title` | `--type-section-title-size` | Section headings |
| `semantic.type.body` | `--type-body-size` | Body copy |
| `semantic.type.meta` | `--type-meta-size` | Metadata, tags, labels |
| `semantic.type.font-body` | `--type-font-body` | Body font family |
| `semantic.type.font-mono` | `--type-font-mono` | Monospace font family |
| `semantic.type.line-height-tight` | `--type-lh-tight` | Tight leading (headings) |
| `semantic.type.line-height-normal` | `--type-lh-normal` | Normal leading (body) |

### radius — corners

| ID | CSS Var | Purpose |
|---|---|---|
| `semantic.radius.subtle` | `--radius-subtle` | Minimal softening |
| `semantic.radius.default` | `--radius-default` | Standard component radius |
| `semantic.radius.pill` | `--radius-pill` | Fully rounded (tags, toggles) |
| `semantic.radius.round` | `--radius-round` | Circles (avatars) |

### elevation — depth

| ID | CSS Var | Purpose |
|---|---|---|
| `semantic.elevation.flat` | `--elevation-flat` | No shadow |
| `semantic.elevation.raised` | `--elevation-raised` | Card, hover-up |
| `semantic.elevation.overlay` | `--elevation-overlay` | Popover, dropdown |
| `semantic.elevation.modal` | `--elevation-modal` | Dialog |

### motion — animation

| ID | CSS Var | Purpose |
|---|---|---|
| `semantic.motion.fast` | `--motion-fast` | Micro-interactions (hover fades) |
| `semantic.motion.medium` | `--motion-medium` | Transitions (menu open) |
| `semantic.motion.slow` | `--motion-slow` | Large motion (modal entry) |
| `semantic.motion.ease-out` | `--motion-ease-out` | Deceleration curve |
| `semantic.motion.ease-in-out` | `--motion-ease-in-out` | Symmetric curve |

### domain — source-specific roles

Some exports have domain-specific tokens that don't map to the canonical vocabulary. Preserve them under `semantic.domain.<name>`. Example from the justinrich.ai fixture:

| ID | CSS Var | Purpose |
|---|---|---|
| `semantic.domain.thinking-tint` | `--domain-thinking-tint` | "Thinking" block background |
| `semantic.domain.thinking-accent` | `--domain-thinking-accent` | "Thinking" block accent |

Domain tokens are fine — they become part of the spec and drive real theming. They're flagged in `gaps.json` only if unused or duplicative.

## Derivation algorithm (Track A + synthesizer)

### Step 1 — Track A proposes (agent, probabilistic)

After extracting concept tokens, Track A emits `foundations.json._semanticProposal`:

```jsonc
{
  "_semanticProposal": [
    {
      "id": "semantic.surface.page",
      "category": "surface",
      "purpose": "Base page background",
      "mappings": {
        "light": { "conceptRef": "color.n-0" },
        "dark":  { "conceptRef": "color.n-0" }
      },
      "evidence": [
        { "file": "tokens.css", "line": 42,
          "context": "--bg: var(--n-0);" }
      ]
    }
  ]
}
```

Evidence ties every proposed semantic token to specific source lines that justify the naming.

### Step 2 — Synthesizer validates (deterministic)

Orchestrator code processes the proposal:

1. For each proposed semantic token:
   - Verify every `conceptRef` resolves in `foundations.json`
   - Verify the `id` pattern `^semantic\.(surface|text|border|accent|state|spacing|type|radius|elevation|motion|domain)\.` — otherwise reject
   - Dedupe by `id` — keep the one with the most evidence; log conflict in gaps
2. Detect **unpromoted concept tokens**:
   - For each concept token in `foundations.json`, check whether any semantic token aliases it
   - If no alias AND the concept token is referenced by any component → emit gap `unpromoted-concept-token` with severity `warning`
   - If no alias AND no references → emit gap `orphan-concept-token` with severity `info`
3. Write `semantic-tokens.json` (valid) + `gaps.json` entries (invalid)

### Step 3 — Synthesizer rewrites refs (deterministic)

For every `tokenRef` in `atoms.json`, `molecules.json`, `organisms.json`, `templates.json`, `pages.json`:

```
if tokenRef starts with "color."/"space."/"type."/... (concept):
  find semantic token whose mappings.{theme}.conceptRef matches
  if found: rewrite tokenRef to semantic.id
  if multiple semantic aliases match: use the most specific one
    (prefer accent.primary over accent.secondary-subtle, etc.)
  if none: leave as concept ref + log gap `concept-only-ref`
```

Concept refs remain valid — they're just less flexible.

### Step 4 — Synthesizer emits theme.css

Pure string emit from `semantic-tokens.json`:

```css
/* Generated by design-deconstruct v2 — DO NOT EDIT BY HAND */
:root {
  --surface-page: /*[light]*/ #f7f6f3;
  --accent-primary: /*[light]*/ #8c3a28;
  --spacing-between: 32px;
  /* ... */
}

[data-theme="dark"] {
  --surface-page: /*[dark]*/ #151513;
  --accent-primary: /*[dark]*/ #c85541;
}
```

Comments include the theme key so humans can scan the mapping without opening `semantic-tokens.json`.

## Using semantic tokens in component specs

Before (v1 — concept refs):
```jsonc
{
  "id": "atom.button",
  "variants": [{
    "name": "primary",
    "states": {
      "default": {
        "bg": { "tokenRef": "color.p-400" },
        "fg": { "tokenRef": "color.n-0" }
      },
      "hover": {
        "bg": { "tokenRef": "color.p-500" }
      }
    }
  }]
}
```

After (v2 — semantic refs):
```jsonc
{
  "id": "atom.button",
  "variants": [{
    "name": "primary",
    "states": {
      "default": {
        "bg": { "tokenRef": "semantic.accent.primary" },
        "fg": { "tokenRef": "semantic.accent.primary-fg" }
      },
      "hover": {
        "bg": { "tokenRef": "semantic.accent.primary-hover" }
      }
    }
  }]
}
```

Implementers consuming the spec see "this uses the primary accent" — clear intent. Theme change = swap `semantic.accent.primary` → different concept token. No component file edited.

## When NOT to promote a concept token

- The concept token is used in a single place with a literal purpose that doesn't fit any category (e.g., a specific branding element). Keep it as domain or leave concept-only.
- The concept token is a *primitive in a ramp* (e.g., `color.n-50` through `color.n-900`) not used directly but only referenced by aliases. Don't emit a semantic for each ramp step — let the ramp stay concept-only.
- The concept token is a "raw input" to another semantic token (e.g., `color.p-400` is referenced only by `semantic.accent.primary`). Keep concept, alias via semantic.

## Mapping the justinrich.ai fixture (worked example)

| Concept | Observed use | Semantic |
|---|---|---|
| `color.n-0` | Page bg, on-accent text | `semantic.surface.page`, `semantic.accent.primary-fg` |
| `color.n-50` | Subtle surfaces | `semantic.surface.soft` |
| `color.n-100` | Divider hover, hairline | `semantic.border.subtle` |
| `color.n-200` | Default borders | `semantic.border.default` |
| `color.n-400` | Captions, status-paused dot | `semantic.text.faint`, `semantic.state.neutral.dot` |
| `color.n-500` | Secondary text | `semantic.text.muted` |
| `color.n-600` | Body | `semantic.text.body` |
| `color.n-700` | Headings | `semantic.text.heading` |
| `color.p-400` | Accent / brick | `semantic.accent.primary` |
| `color.p-500` | Link, hover/active | `semantic.text.link`, `semantic.accent.primary-hover` |
| `color.p-600` | Pressed | `semantic.accent.primary-pressed` |
| `color.s-active` | Active status dot | `semantic.state.success.dot` |
| `color.s-shipped` | Shipped status dot | `semantic.state.info.dot` |
| `color.thinking-tint` | Thinking block bg | `semantic.domain.thinking-tint` |
| `space.sp-1` | Micro gaps | `semantic.spacing.micro` |
| `space.sp-2` | Entry internal | `semantic.spacing.internal` |
| `space.sp-3` | Between entries | `semantic.spacing.between` |
| `space.sp-4` | Page sections | `semantic.spacing.section` |
| `type.fs-1` | Page titles | `semantic.type.page-title` |
| `type.fs-2` | Entry titles | `semantic.type.section-title` |
| `type.fs-3` | Body | `semantic.type.body` |
| `type.fs-4` | Metadata | `semantic.type.meta` |
| `radius.r-1` | Tags | `semantic.radius.subtle` |
| `radius.r-2` | Components | `semantic.radius.default` |

Concept tokens that stayed concept-only: the intermediate neutrals (`n-300`, `n-800`, `n-900`) — they exist in the ramp but aren't referenced directly by components. Fine to leave.

## Gap types for the semantic layer

| Gap kind | Severity | Meaning |
|---|---|---|
| `unpromoted-concept-token` | warning | Concept token used by components but no semantic alias — promote candidate |
| `orphan-concept-token` | info | Concept token never referenced by any component or semantic |
| `concept-only-ref` | info | Component ref uses concept directly; semantic rewrite failed |
| `semantic-collision` | warning | Two proposed semantic tokens have the same id |
| `semantic-circular` | error | Semantic token's aliasFor chain loops |
| `semantic-dangling` | error | Semantic token's conceptRef does not exist in foundations |
