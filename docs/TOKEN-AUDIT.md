# TOKEN-AUDIT

Grep rules and reject-and-regen policy run at the end of every phase (after subagent returns, before TaskUpdate → completed).

## WHAT WE REJECT

| Pattern | Reason |
|---|---|
| `#[0-9a-fA-F]{3,6}\b` in any atom/molecule/organism/view CSS | Raw hex literals — colors must come from `--{surface/text/border/accent/status}-*` |
| `font-size:\s*\d+(\.\d+)?(px\|em\|rem\|%)` | Numeric font-size — use `var(--font-size-*)` |
| `font-weight:\s*\d{3}` | Numeric font-weight — use `var(--font-weight-*)` |
| `line-height:\s*[0-9.]+(?!\s*var)` | Numeric line-height — use `var(--line-height-*)` |
| `letter-spacing:\s*[\-]?[0-9.]+em` | Numeric tracking — use `var(--tracking-*)` |
| Raw `\d+(?:\.\d+)?px` in padding/margin/gap (excluding border-radius/width/stroke-width) | Use `var(--space-*)` |

## WHAT WE ALLOW

- `1px` when referring to SVG `stroke-width` — SVG units differ from CSS
- Percentage widths (`width: 100%`, `max-width: 90%`) — content-adaptive, not design tokens
- `ch` units on `max-width` (e.g. `max-width: 70ch`) — content-adaptive
- `aspect-ratio: 16/9` and similar ratio literals — structural, not stylistic
- Values inside CSS `calc()` that use `var(...)` — computation wrapping tokens is fine
- Preview-chrome literals inside `atoms/_preview.css` that explicitly pin theme backgrounds (e.g. `[data-theme="light"] { background: #f7f6f3; }`) — required for dual-theme rendering in a single HTML document; flagged once in the file header comment
- SVG path data (`d="M …"`, viewBox coords) — geometry, not CSS
- Animation timing like `1.4s`, `400ms` when no motion token covers the case — emit TOKEN_GAP first; only accept literal if the orchestrator's own grep finds no matching `var(--motion-duration-*)`

## AUDIT ALGORITHM

```
FOR each file in the newly-written component:
  IF file matches atoms/*/*.html, molecules/*/*.html, organisms/*/*.html, views/*/*.html,
     OR is a bundle file (molecules/_atoms.css, organisms/_molecules.css, etc.):
    Extract <style> blocks (HTML) or whole file (CSS).
    FOR each line:
      IF line defines a CSS declaration AND line contains a banned pattern
         AND line does NOT reference var(...) in that same declaration
         AND line is not inside an inline style="" demo attribute:
        RECORD violation: {file, line, pattern, text}.

IF violations list is non-empty:
  Group violations by file.
  Re-dispatch the component's subagent with:
    "AUDIT FAILURE: the following lines contain hardcoded values that must
     resolve to tokens. Fix them and re-write the file. If a needed token
     does not exist, emit TOKEN_GAP at the top of your reply."
    + (per file) a numbered list of offending lines with line numbers.
  Cap at 2 re-dispatches per component. After 2, log to manifest.json "gaps"
  and surface in final report.
```

## PYTHON SNIPPET (orchestrator-owned)

```python
import re, os

BANNED_PATTERNS = [
    (r'#[0-9a-fA-F]{6}\b',                          'hex-literal'),
    (r'font-size:\s*\d+(?:\.\d+)?(?:px|em|rem|%)',  'font-size-literal'),
    (r'font-weight:\s*\d{3}\b',                     'font-weight-literal'),
    (r'line-height:\s*[0-9.]+\s*[;}]',              'line-height-literal'),
    (r'letter-spacing:\s*[\-]?[0-9.]+em',           'letter-spacing-literal'),
]

SAFE_CONTEXTS = ['stroke-width', 'viewBox', 'style="', "style='"]

def audit_file(path):
    violations = []
    with open(path) as f: src = f.read()
    blocks = re.findall(r'<style>(.*?)</style>', src, flags=re.DOTALL) \
             if path.endswith('.html') else [src]
    for block in blocks:
        for i, line in enumerate(block.split('\n'), 1):
            if any(s in line for s in SAFE_CONTEXTS): continue
            if 'var(--' in line:                        continue  # safe if any var() present
            for pattern, tag in BANNED_PATTERNS:
                if re.search(pattern, line):
                    violations.append((path, i, tag, line.strip()))
    return violations
```

Extend with the padding/margin/gap grep once you have a list of all allowed token aliases (grep for `\d+px` NOT preceded by `var(--` and NOT within a SAFE_CONTEXTS line).

## WHEN TO EXTEND TOKENS vs. REJECT-AND-REGEN

- Subagent emitted `TOKEN_GAP:` → orchestrator extends `tokens/tokens.css` and continues.
- Audit found a literal without an accompanying `TOKEN_GAP:` → regression; re-dispatch.

A clean phase completion means:
- Zero audit violations across all newly-written files in that phase AND
- Any `TOKEN_GAP:` escapes were resolved (token added, file re-verified clean).
