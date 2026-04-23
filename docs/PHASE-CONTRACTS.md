# PHASE-CONTRACTS

Per-phase subagent dispatch prompts and contracts. The orchestrator substitutes `<<PLACEHOLDERS>>` at dispatch time.

All phases share a common preamble (Quality Bar + Escape Protocols). Each phase adds its specific inputs, outputs, and constraints.

---

## SHARED PREAMBLE (prepended to every phase prompt)

```
You are a frontend-designer subagent working on ONE component at a time in the
"design-deconstruct" workflow. The orchestrator calls you sequentially.

NON-NEGOTIABLE QUALITY BAR:
  1. Zero hex literals — every color is var(--{semantic}).
  2. Zero numeric font-size / font-weight / line-height / letter-spacing.
     Use var(--font-size-*), var(--font-weight-*), var(--line-height-*), var(--tracking-*).
  3. Zero raw px in padding / margin / gap — use var(--space-*) scale.
  4. No placeholder content — render real markup with real content from the concept.
  5. Every variant × state rendered — use .is-{name} forced classes.
  6. Self-contained HTML — only link tokens.css, _preview.css, (and _atoms.css for
     molecules+). No CDNs. No build step.
  7. Composition purity — never redefine a lower-layer atom's styles.

ESCAPE PROTOCOLS (emit at the top of your reply, BEFORE any file writes):

  TOKEN_GAP: --{key} := {value} ({reason})
    When a needed semantic token doesn't exist in tokens/tokens.css.
    Example:
      TOKEN_GAP: --surface-card-hover := #f0efeb (hover bg for entry cards)

  VARIANT_REQUEST: atom|molecule|organism={name} variant={class-name} recipe={css-rules}
    When you need a lower-layer variant that doesn't exist.
    Example:
      VARIANT_REQUEST: atom=button variant=atom-button--danger
        recipe={ background: var(--status-error); color: var(--text-on-brand); }

After any escapes, proceed with your primary task. The orchestrator handles the
gap/variant, then either asks you to retry (if your output depended on it) or
moves on.
```

---

## PHASE 1 — TOKENS

```
TASK: Extract the complete token system from the concept HTML.

INPUT:
  • Concept HTML path:   <<CONCEPT_PATH>>
  • Decoded source files (if concept was bundled):  <<DECODED_SOURCES>>
  • Output directory:    <<OUTPUT_DIR>>

DELIVERABLES (write all of these):

  <<OUTPUT_DIR>>/tokens/tokens.css
    Two-tier structure:
      Tier 1 (private): --_{name} primitives by theme, prefixed with _
      Tier 2 (semantic): --{role}-{variant} aliases identical keys across themes
    Support both themes via data-theme="light" (default) and [data-theme="dark"].
    Organize by role: surface · text · border · accent · status · elevation
                      motion · radius · space · type · control · icon · ornament.

  <<OUTPUT_DIR>>/tokens/theme.light.json
  <<OUTPUT_DIR>>/tokens/theme.dark.json
    JSON mirrors of the semantic layer. Grouped by role.

  <<OUTPUT_DIR>>/tokens/theme.schema.json
    JSONSchema Draft 2020-12. MUST require identical keys across both themes.

  <<OUTPUT_DIR>>/typography/fonts.css
    @font-face (or @import) + the font stack as --font-mono / --font-sans.
    Feature settings if the design uses them.

  <<OUTPUT_DIR>>/typography/type-modules.css
    Named modules: .type-h1 / .type-h2 / .type-title / .type-body / .type-meta /
                   .type-label / .type-code
    Each module baked with: size + weight + line-height + tracking + case.
    Every value is var(--font-size-*), var(--font-weight-*), var(--line-height-*),
    var(--tracking-*). No literals.

  <<OUTPUT_DIR>>/tokens.html
    Rendered primitives index. Two-theme panes side-by-side per section:
      typography · surface · text · border · accent · status · elevation ·
      spacing · radius · motion · layout/stroke.
    Each section uses data-theme="light" and data-theme="dark" panes so both
    themes render in a single document. Forward reference to load order:
      <link rel="stylesheet" href="./typography/fonts.css">
      <link rel="stylesheet" href="./tokens/tokens.css">
      <link rel="stylesheet" href="./typography/type-modules.css">

SEMANTIC VOCABULARY:
  Load docs/SEMANTIC-TOKENS.md for canonical role names.

SCHEMAS:
  Load docs/OUTPUT-SCHEMA.md § theme.schema.json for JSON structure.

BUNDLED-CONCEPT DECODE SNIPPET (if needed):
  The concept HTML may embed a bundler template + manifest of base64+gzip blobs.
  If you see <script type="__bundler/manifest"> and <script type="__bundler/template">,
  use this Python snippet to decode:

    import json, base64, gzip
    with open(<<CONCEPT_PATH>>) as f: content = f.read()
    m_start = content.find('<script type="__bundler/manifest">')
    m_end = content.find('</script>', m_start)
    raw = content[m_start:m_end][content[m_start:m_end].index('>')+1:]
    manifest = json.loads(raw)
    for uuid, entry in manifest.items():
        if entry['mime'] in ('text/javascript','application/javascript','text/jsx'):
            data = base64.b64decode(entry['data'])
            if entry.get('compressed'): data = gzip.decompress(data)
            open(f'/tmp/bundle_{uuid[:8]}.js', 'wb').write(data)

  Then read the JS files to extract the actual design CSS/component code.

OUTPUT RULES:
  • Every token name describes USE, not value. NEVER use color names or px
    in keys ("white", "1px" are banned; "surface-canvas", "stroke-hairline" are good).
  • Keys in light.json and dark.json MUST be identical (only values differ).
  • tokens.html renders every primitive visibly in BOTH themes — no placeholders.
```

---

## PHASE 2 — ATOMS

```
TASK: Deconstruct ONE atom from the concept HTML.

INPUT:
  • Atom name:          <<ATOM_NAME>>
  • Concept HTML path:  <<CONCEPT_PATH>>
  • Concept excerpt:    <<CONCEPT_EXCERPT>>    (relevant source lines)
  • Tokens path:        <<OUTPUT_DIR>>/tokens/tokens.css
  • Type modules path:  <<OUTPUT_DIR>>/typography/type-modules.css
  • Preview frame:      <<OUTPUT_DIR>>/atoms/_preview.css

DELIVERABLES (write both):

  <<OUTPUT_DIR>>/atoms/<<ATOM_NAME>>/README.md
    Sections in this order:
      # <<ATOM_NAME>>
      ## Purpose
      ## Anatomy          HTML snippet
      ## Variants         table
      ## States            default/hover/active/focus/disabled/etc.
      ## Token recipe     table: property → token
      ## Accessibility    ARIA / keyboard rules
      ## Atom-local constants   (only if truly atom-specific; call out reason)

  <<OUTPUT_DIR>>/atoms/<<ATOM_NAME>>/<<ATOM_NAME>>.html
    Full preview page. Imports in this order:
      <link rel="stylesheet" href="../../typography/fonts.css">
      <link rel="stylesheet" href="../../tokens/tokens.css">
      <link rel="stylesheet" href="../../typography/type-modules.css">
      <link rel="stylesheet" href="../_preview.css">
    Inside <style>, define ONLY .atom-<<ATOM_NAME>> rules (and
    preview-local helpers). Use .is-hover/.is-active/.is-focus/.is-disabled
    classes so every state renders deterministically.
    Structure: header.plate + pegboard + variant sections + recipe table + footer.

CONSTRAINTS:
  • Atoms are INDIVISIBLE — no composition from other atoms.
  • If you find yourself importing another atom class, STOP — what you're
    describing is a molecule, not an atom. Narrow the scope.
  • Reference only theme tokens. No hex, no raw px/em in typography declarations,
    no font-weight numerics.
  • If a needed token doesn't exist: emit TOKEN_GAP: --{key} := {value} ({reason})
    at the top of your reply.

ATOM INVENTORY (single-purpose candidates from this design):
  button · link · tag · filter-pill · status-dot · glyph · divider ·
  kbd · timestamp · focus-ring · spinner · skeleton  (adjust per concept)
```

---

## PHASE 3 — MOLECULES

```
TASK: Deconstruct ONE molecule from the concept HTML.

INPUT:
  • Molecule name:      <<MOLECULE_NAME>>
  • Concept HTML path:  <<CONCEPT_PATH>>
  • Concept excerpt:    <<CONCEPT_EXCERPT>>
  • Tokens + typography paths (as Phase 2)
  • Preview frame:      <<OUTPUT_DIR>>/atoms/_preview.css
  • Atoms bundle:       <<OUTPUT_DIR>>/molecules/_atoms.css
  • Atoms index:        <<OUTPUT_DIR>>/atoms/README.md

DELIVERABLES (write both):

  <<OUTPUT_DIR>>/molecules/<<MOLECULE_NAME>>/README.md
    Same structure as Phase 2 README, PLUS:
      ## Atoms used   table: atom · role
    The atoms-used table is REQUIRED — it documents the composition explicitly.

  <<OUTPUT_DIR>>/molecules/<<MOLECULE_NAME>>/<<MOLECULE_NAME>>.html
    Full preview page. Imports in this order:
      <link rel="stylesheet" href="../../typography/fonts.css">
      <link rel="stylesheet" href="../../tokens/tokens.css">
      <link rel="stylesheet" href="../../typography/type-modules.css">
      <link rel="stylesheet" href="../../atoms/_preview.css">
      <link rel="stylesheet" href="../_atoms.css">
    Inside <style>, define ONLY .mol-<<MOLECULE_NAME>> rules (molecule layout
    and composition) and preview helpers. NEVER redefine .atom-* styling —
    atoms are bundled in _atoms.css.

CONSTRAINTS:
  • Compose atoms by class name.
  • Molecule rules only add layout glue (display/flex/gap/grid) and
    molecule-specific typography/color overrides that resolve to tokens.
  • Any value must resolve to a theme token or come from a composed atom.
  • If a needed atom variant doesn't exist: emit
      VARIANT_REQUEST: atom=<<atom-name>> variant=<<class>> recipe=<<css-rules>>
    The orchestrator will update the atom + regen quartet, then resume you.
  • If a needed token doesn't exist: emit TOKEN_GAP as per Phase 2.

MOLECULE INVENTORY (2+ atom compositions from this design):
  entry-type-badge · status-badge · tag-list · filter-group · breadcrumb ·
  kbd-chord · inline-meta-list · external-link · nav-link · section-header ·
  meta-gutter · sandbox-chrome  (adjust per concept)
```

---

## PHASE 4 — ORGANISMS

```
TASK: Deconstruct ONE organism from the concept HTML.

INPUT:
  • Organism name:      <<ORGANISM_NAME>>
  • Concept HTML path:  <<CONCEPT_PATH>>
  • Concept excerpt:    <<CONCEPT_EXCERPT>>
  • Tokens + typography + atoms + molecules paths
  • Preview frame:      <<OUTPUT_DIR>>/atoms/_preview.css
  • Atoms bundle:       <<OUTPUT_DIR>>/molecules/_atoms.css
  • Molecules bundle:   <<OUTPUT_DIR>>/organisms/_molecules.css  (NEW per phase 4)
  • Indexes:            atoms/README.md, molecules/README.md

DELIVERABLES (write both):

  <<OUTPUT_DIR>>/organisms/<<ORGANISM_NAME>>/README.md
    Same structure as Phase 3, PLUS:
      ## Composes    table: molecule|atom · role
    Lists every molecule and atom the organism consumes.

  <<OUTPUT_DIR>>/organisms/<<ORGANISM_NAME>>/<<ORGANISM_NAME>>.html
    Full preview page. Imports add:
      <link rel="stylesheet" href="../_molecules.css">    (bundle of molecule rules)
    Inside <style>, .org-<<ORGANISM_NAME>> rules only (layout/glue).

CONSTRAINTS:
  • Compose molecules primarily, atoms where appropriate (e.g. a divider
    between sections doesn't need a molecule wrapper).
  • NEVER redefine atom or molecule styling.
  • VARIANT_REQUESTs may target atom OR molecule. Cascade rules: docs/REGEN-CASCADE.md.

ORGANISM INVENTORY (page-section units from this design):
  nav · filter-bar · feed-entry · feed-entry-thinking · project-row ·
  footer · desktop-frame · entry-detail-header  (adjust per concept)
```

---

## PHASE 5 — VIEWS

```
TASK: Deconstruct ONE view (full page template) from the concept HTML.

INPUT:
  • View name:          <<VIEW_NAME>>
  • Concept HTML path:  <<CONCEPT_PATH>>
  • Concept excerpt:    <<CONCEPT_EXCERPT>>
  • Tokens + typography + atoms + molecules + organisms paths
  • Bundles: _preview.css, _atoms.css, _molecules.css, _organisms.css

DELIVERABLES (write both):

  <<OUTPUT_DIR>>/views/<<VIEW_NAME>>/README.md
    Same structure as Phase 4, PLUS:
      ## Composes    table: organism|molecule|atom · role in view
      ## Responsive   breakpoints and layout shifts (if any)

  <<OUTPUT_DIR>>/views/<<VIEW_NAME>>/<<VIEW_NAME>>.html
    Full page mock. Imports add:
      <link rel="stylesheet" href="../_organisms.css">    (bundle of organism rules)
    Inside <style>, .view-<<VIEW_NAME>> rules only (page-level layout).

CONSTRAINTS:
  • Compose organisms primarily, dropping to molecules/atoms only where an
    organism doesn't cover the need.
  • VARIANT_REQUESTs may target any lower layer. Max cascade depth 3.
  • Views MUST use stacked vertical layout — each variant in its own
    full-width <section>. Do NOT use .two-up or side-by-side grids.
    Light and dark theme panes stack vertically as separate sections,
    not side-by-side columns.
  • Include @media queries for responsive breakpoints:
    Desktop: default (max-width per layout token)
    Mobile: max-width 375px — single column, stacked nav, full-width entries
    Use var(--layout-*) tokens (add to tokens.css via TOKEN_GAP if missing).

VIEW INVENTORY (full page compositions from this design):
  home-feed · projects-index · about · entry-detail · entry-detail-thinking
  (adjust per concept)
```

---

## ORCHESTRATOR REGEN HANDLER (for VARIANT_REQUEST)

When the orchestrator receives `VARIANT_REQUEST: atom=<name> variant=<class> recipe=<rules>`:

```
Dispatch a TARGETED frontend-designer subagent with this prompt:

  TASK: Add a new variant to an EXISTING atom.

  ATOM: <<ATOM_NAME>>
  NEW VARIANT CLASS: <<VARIANT_CLASS>>
  PROPOSED RECIPE: <<RECIPE>>
  REASON: <<REASON_FROM_REQUESTOR>>

  DELIVERABLES:
    1. Update atoms/<<ATOM_NAME>>/<<ATOM_NAME>>.html:
       • Add the new variant's CSS rules inside the existing <style>
         block (after existing variant selectors).
       • Add a new <section class="variant"> rendering the new variant
         in both theme panes with all applicable states.
    2. Update atoms/<<ATOM_NAME>>/README.md:
       • Add a row to the Variants table.
       • Update the Recipe table with the new variant's token refs.
       • If the recipe reveals a token gap, emit TOKEN_GAP.

  CONSTRAINTS:
    • Preserve every existing variant exactly — additive only.
    • The new variant must reference theme tokens (no literals).
    • If recipe has a literal that should be a token, emit TOKEN_GAP instead.

After the subagent returns, the orchestrator:
  1. Re-renders atoms/<<ATOM_NAME>>/<<ATOM_NAME>>.pdf and .png.
  2. Appends the variant's CSS rules to molecules/_atoms.css (or
     organisms/_molecules.css if the request was for a molecule).
  3. Resumes the paused phase.
```

Analogous templates for VARIANT_REQUEST on molecules / organisms (substitute the layer's folder + bundle path).
