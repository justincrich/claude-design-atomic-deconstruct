# Output JSON Schemas — design-deconstruct v1

JSON Schema Draft 2020-12 definitions for every emitted file.

At runtime, the synthesizer copies each schema below into `<output>/schemas/` and validates every emitted layer file against its schema. Implementer agents can re-validate with any standard JSON Schema validator (ajv, jsonschema, etc.).

## Global conventions

- **IDs**: `{layer}.{kebab-slug}` — e.g. `color.p-400`, `atom.button`, `page.home`.
- **References**: `"tokenRef": "color.p-400"`, `"atomRef": "atom.button"`, `"templateRef": "template.sidebar-main"`.
- **Theme-aware values**: `{ "values": { "light": "#f7f6f3", "dark": "#151513" } }`.
- **Source refs**: `{ "file": "tokens.css", "lines": [7, 7] }` (single line = `[n, n]`).
- **Nullable**: states that don't apply = `null`; states not yet documented = missing from object (plus entry in `stateCoverage.missing`).

---

## 1. design-system.schema.json (top-level)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://brain/schemas/design-system/v1/design-system.schema.json",
  "title": "Design System (top-level manifest)",
  "type": "object",
  "required": ["meta", "counts", "index"],
  "properties": {
    "meta": {
      "type": "object",
      "required": ["generatedAt", "generator", "version", "source", "themes", "bundleType"],
      "properties": {
        "generatedAt": { "type": "string", "format": "date-time" },
        "generator":   { "const": "design-deconstruct" },
        "version":     { "type": "string", "pattern": "^\\d+\\.\\d+\\.\\d+$" },
        "source": {
          "type": "object",
          "required": ["path", "fileCount"],
          "properties": {
            "path":        { "type": "string" },
            "fileCount":   { "type": "integer", "minimum": 1 },
            "manifestRef": { "type": "string" }
          }
        },
        "platformHint": { "type": ["string", "null"], "enum": ["web", "mobile", "desktop", "tui", "wearable", null] },
        "themes":       { "type": "array", "items": { "type": "string" }, "minItems": 1 },
        "bundleType":   { "enum": ["claude-design", "handoff-bundle", "figma", "generic"] }
      }
    },
    "counts": {
      "type": "object",
      "required": ["tokens", "atoms", "molecules", "organisms", "templates", "pages", "flows", "gaps"],
      "properties": {
        "tokens":    { "type": "integer", "minimum": 0 },
        "atoms":     { "type": "integer", "minimum": 0 },
        "molecules": { "type": "integer", "minimum": 0 },
        "organisms": { "type": "integer", "minimum": 0 },
        "templates": { "type": "integer", "minimum": 0 },
        "pages":     { "type": "integer", "minimum": 0 },
        "flows":     { "type": "integer", "minimum": 0 },
        "gaps":      { "type": "integer", "minimum": 0 }
      }
    },
    "index": {
      "type": "object",
      "properties": {
        "foundations":     { "type": "string" },
        "primitives":      { "type": "string" },
        "atoms":           { "type": "string" },
        "molecules":       { "type": "string" },
        "organisms":       { "type": "string" },
        "templates":       { "type": "string" },
        "pages":           { "type": "string" },
        "useCases":        { "type": "string" },
        "tokenMap":        { "type": "string" },
        "componentIndex":  { "type": "string" },
        "dependencyGraph": { "type": "string" },
        "stateCoverage":   { "type": "string" },
        "gaps":            { "type": "string" },
        "manifest":        { "type": "string" }
      }
    }
  }
}
```

---

## 2. manifest.schema.json

```json
{
  "$id": "https://brain/schemas/design-system/v1/manifest.schema.json",
  "type": "object",
  "required": ["generatedAt", "bundleType", "files", "entryPoints"],
  "properties": {
    "generatedAt": { "type": "string", "format": "date-time" },
    "bundleType":  { "enum": ["claude-design", "handoff-bundle", "figma", "generic"] },
    "files": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["path", "sha256", "loc", "bytes", "kind"],
        "properties": {
          "path":   { "type": "string" },
          "sha256": { "type": "string", "pattern": "^[a-f0-9]{64}$" },
          "loc":    { "type": "integer", "minimum": 0 },
          "bytes":  { "type": "integer", "minimum": 0 },
          "kind":   { "enum": ["source-html", "source-component", "source-style", "context-doc", "asset", "other"] }
        }
      }
    },
    "entryPoints": { "type": "array", "items": { "type": "string" } }
  }
}
```

---

## 3. foundations.schema.json

```json
{
  "$id": "https://brain/schemas/design-system/v1/foundations.schema.json",
  "type": "object",
  "required": ["meta", "colors", "typography", "spacing", "radii", "elevation", "motion", "iconography"],
  "$defs": {
    "sourceRef": {
      "type": "object",
      "required": ["file", "lines"],
      "properties": {
        "file":  { "type": "string" },
        "lines": { "type": "array", "items": { "type": "integer" }, "minItems": 2, "maxItems": 2 }
      }
    },
    "themedValue": {
      "type": "object",
      "additionalProperties": { "type": ["string", "number"] },
      "minProperties": 1
    },
    "colorToken": {
      "type": "object",
      "required": ["id", "values", "role", "sourceRef"],
      "properties": {
        "id":        { "type": "string", "pattern": "^color\\." },
        "values":    { "$ref": "#/$defs/themedValue" },
        "role":      { "type": "string" },
        "aliasFor":  { "type": "string" },
        "usedBy":    { "type": "array", "items": { "type": "string" } },
        "sourceRef": { "$ref": "#/$defs/sourceRef" }
      }
    },
    "scaleToken": {
      "type": "object",
      "required": ["id", "role"],
      "properties": {
        "id":   { "type": "string" },
        "px":   { "type": "number" },
        "rem":  { "type": "number" },
        "role": { "type": "string" }
      }
    }
  },
  "properties": {
    "meta": {
      "type": "object",
      "required": ["themes", "sources"],
      "properties": {
        "themes":  { "type": "array", "items": { "type": "string" } },
        "sources": { "type": "array", "items": { "type": "string" } }
      }
    },
    "colors": {
      "type": "object",
      "properties": {
        "neutrals": { "type": "array", "items": { "$ref": "#/$defs/colorToken" } },
        "primary":  { "type": "array", "items": { "$ref": "#/$defs/colorToken" } },
        "accent":   { "type": "array", "items": { "$ref": "#/$defs/colorToken" } },
        "status":   { "type": "object", "additionalProperties": { "$ref": "#/$defs/colorToken" } },
        "aliases":  {
          "type": "object",
          "additionalProperties": {
            "type": "object",
            "required": ["aliasFor"],
            "properties": { "aliasFor": { "type": "string" } }
          }
        }
      }
    },
    "typography": {
      "type": "object",
      "required": ["fonts", "scale", "lineHeights", "weights"],
      "properties": {
        "fonts": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["id", "stack", "role"],
            "properties": {
              "id":       { "type": "string", "pattern": "^type\\.font-" },
              "stack":    { "type": "array", "items": { "type": "string" } },
              "role":     { "type": "string" },
              "features": { "type": "array", "items": { "type": "string" } }
            }
          }
        },
        "scale": {
          "type": "array",
          "items": { "$ref": "#/$defs/scaleToken" }
        },
        "lineHeights": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["id", "value", "role"],
            "properties": {
              "id":    { "type": "string", "pattern": "^type\\.lh-" },
              "value": { "type": "number" },
              "role":  { "type": "string" }
            }
          }
        },
        "weights": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["id", "value"],
            "properties": {
              "id":    { "type": "string", "pattern": "^type\\.weight-" },
              "value": { "type": "integer", "minimum": 100, "maximum": 900 }
            }
          }
        },
        "letterSpacing": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "id":    { "type": "string" },
              "em":    { "type": "number" },
              "role":  { "type": "string" }
            }
          }
        }
      }
    },
    "spacing":   { "type": "array", "items": { "$ref": "#/$defs/scaleToken" } },
    "radii":     { "type": "array", "items": { "$ref": "#/$defs/scaleToken" } },
    "elevation": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "value", "role"],
        "properties": {
          "id":    { "type": "string", "pattern": "^shadow\\." },
          "value": { "type": "string", "description": "CSS box-shadow syntax or equivalent abstract" },
          "role":  { "type": "string" }
        }
      }
    },
    "motion": {
      "type": "object",
      "required": ["durations", "easings"],
      "properties": {
        "durations": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["id", "ms", "role"],
            "properties": {
              "id":   { "type": "string", "pattern": "^motion\\.dur-" },
              "ms":   { "type": "number", "minimum": 0 },
              "role": { "type": "string" }
            }
          }
        },
        "easings": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["id", "curve", "role"],
            "properties": {
              "id":    { "type": "string", "pattern": "^motion\\.ease-" },
              "curve": { "type": "string" },
              "role":  { "type": "string" }
            }
          }
        }
      }
    },
    "iconography": {
      "type": "object",
      "properties": {
        "sizes":        { "type": "array", "items": { "$ref": "#/$defs/scaleToken" } },
        "strokeWidths": { "type": "array", "items": { "$ref": "#/$defs/scaleToken" } },
        "library":      { "type": "string" }
      }
    },
    "anonymousLiterals": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["value", "occurrences", "kind"],
        "properties": {
          "value":            { "type": "string" },
          "kind":             { "enum": ["color", "space", "radius", "duration", "other"] },
          "occurrences":      { "type": "integer", "minimum": 1 },
          "elevateCandidate": { "type": "string" },
          "sourceRefs":       { "type": "array", "items": { "$ref": "#/$defs/sourceRef" } }
        }
      }
    }
  }
}
```

---

## 4. atom.schema.json

```json
{
  "$id": "https://brain/schemas/design-system/v1/atom.schema.json",
  "type": "object",
  "required": ["atoms"],
  "$defs": {
    "tokenRef":     { "type": "object", "required": ["tokenRef"], "properties": { "tokenRef": { "type": "string" } } },
    "literalValue": { "type": "object", "required": ["literal"], "properties": { "literal": { "type": ["string", "number"] } } },
    "valueRef":     { "oneOf": [ { "$ref": "#/$defs/tokenRef" }, { "$ref": "#/$defs/literalValue" }, { "type": "null" } ] },
    "stateSpec": {
      "type": ["object", "null"],
      "properties": {
        "bg":         { "$ref": "#/$defs/valueRef" },
        "fg":         { "$ref": "#/$defs/valueRef" },
        "border":     { "$ref": "#/$defs/valueRef" },
        "borderWidth":{ "$ref": "#/$defs/valueRef" },
        "radius":     { "$ref": "#/$defs/valueRef" },
        "padding": {
          "type": "object",
          "properties": {
            "x": { "$ref": "#/$defs/valueRef" },
            "y": { "$ref": "#/$defs/valueRef" },
            "t": { "$ref": "#/$defs/valueRef" },
            "r": { "$ref": "#/$defs/valueRef" },
            "b": { "$ref": "#/$defs/valueRef" },
            "l": { "$ref": "#/$defs/valueRef" }
          }
        },
        "typography": {
          "type": "object",
          "properties": {
            "fontRef":   { "type": "string" },
            "sizeRef":   { "type": "string" },
            "weightRef": { "type": "string" },
            "lhRef":     { "type": "string" },
            "weight":    { "type": "integer" }
          }
        },
        "shadow":     { "$ref": "#/$defs/valueRef" },
        "opacity":    { "type": "number", "minimum": 0, "maximum": 1 },
        "transform":  { "type": "string" },
        "cursor":     { "type": "string" },
        "transition": { "type": "object", "properties": { "durationRef": {"type":"string"}, "easingRef": {"type":"string"}, "properties": {"type":"array","items":{"type":"string"}} } }
      }
    }
  },
  "properties": {
    "atoms": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "name", "layer", "variants", "sourceRefs"],
        "properties": {
          "id":    { "type": "string", "pattern": "^atom\\." },
          "name":  { "type": "string" },
          "layer": { "const": "atom" },
          "category": { "enum": ["action", "input", "display", "feedback", "navigation", "misc"] },
          "variants": {
            "type": "array",
            "items": {
              "type": "object",
              "required": ["name", "states", "stateCoverage"],
              "properties": {
                "name":  { "type": "string" },
                "sizes": { "type": "array", "items": { "type": "string" } },
                "states": {
                  "type": "object",
                  "properties": {
                    "default":       { "$ref": "#/$defs/stateSpec" },
                    "hover":         { "$ref": "#/$defs/stateSpec" },
                    "active":        { "$ref": "#/$defs/stateSpec" },
                    "focus":         { "$ref": "#/$defs/stateSpec" },
                    "focus-visible": { "$ref": "#/$defs/stateSpec" },
                    "disabled":      { "$ref": "#/$defs/stateSpec" },
                    "loading":       { "$ref": "#/$defs/stateSpec" },
                    "error":         { "$ref": "#/$defs/stateSpec" },
                    "selected":      { "$ref": "#/$defs/stateSpec" },
                    "checked":       { "$ref": "#/$defs/stateSpec" }
                  }
                },
                "stateCoverage": {
                  "type": "object",
                  "required": ["documented", "missing", "inapplicable"],
                  "properties": {
                    "documented":   { "type": "array", "items": { "type": "string" } },
                    "missing":      { "type": "array", "items": { "type": "string" } },
                    "inapplicable": { "type": "array", "items": { "type": "string" } }
                  }
                }
              }
            }
          },
          "dimensions": {
            "type": "object",
            "properties": {
              "minHeight": { "$ref": "#/$defs/valueRef" },
              "minWidth":  { "$ref": "#/$defs/valueRef" },
              "maxHeight": { "$ref": "#/$defs/valueRef" },
              "maxWidth":  { "$ref": "#/$defs/valueRef" }
            }
          },
          "accessibility": {
            "type": "object",
            "properties": {
              "role":          { "type": "string" },
              "ariaAttrs":     { "type": "array", "items": { "type": "string" } },
              "keyboard":      { "type": "array", "items": { "type": "string" } },
              "focusRing":     { "type": "object", "properties": { "offset": {"type":"string"}, "colorRef": {"type":"string"} } }
            }
          },
          "sourceRefs": { "type": "array", "items": { "$ref": "https://brain/schemas/design-system/v1/foundations.schema.json#/$defs/sourceRef" } },
          "usedBy":     { "type": "array", "items": { "type": "string" } },
          "ambiguous":  { "type": "boolean" },
          "_gaps":      { "type": "array", "items": { "type": "string" } }
        }
      }
    }
  }
}
```

---

## 5. molecule.schema.json / organism.schema.json (shared composition shape)

```json
{
  "$id": "https://brain/schemas/design-system/v1/composition.schema.json",
  "type": "object",
  "properties": {
    "molecules":  { "type": "array", "items": { "$ref": "#/$defs/composition" } },
    "organisms":  { "type": "array", "items": { "$ref": "#/$defs/composition" } }
  },
  "$defs": {
    "composition": {
      "type": "object",
      "required": ["id", "name", "layer", "composedOf", "sourceRefs"],
      "properties": {
        "id":   { "type": "string", "pattern": "^(molecule|organism)\\." },
        "name": { "type": "string" },
        "layer": { "enum": ["molecule", "organism"] },
        "composedOf": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["slot", "ref"],
            "properties": {
              "slot":     { "type": "string" },
              "ref":      { "type": "string", "description": "atomRef | moleculeRef" },
              "props":    { "type": "object" },
              "repeat":   { "type": ["integer", "string"], "description": "number or 'N' for dynamic" },
              "optional": { "type": "boolean" }
            }
          }
        },
        "layout": {
          "type": "object",
          "properties": {
            "direction":   { "enum": ["row", "column", "grid", "stack"] },
            "gap":         { "type": "string" },
            "align":       { "type": "string" },
            "justify":     { "type": "string" },
            "gridColumns": { "type": "string" },
            "wrapping":    { "type": "boolean" }
          }
        },
        "states": {
          "type": "object",
          "additionalProperties": {
            "type": "object",
            "properties": {
              "description": { "type": "string" },
              "overrides":   { "type": "object" }
            }
          }
        },
        "responsive": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["minWidth"],
            "properties": {
              "minWidth":  { "type": ["integer", "string"] },
              "overrides": { "type": "object" }
            }
          }
        },
        "sourceRefs": { "type": "array", "items": { "$ref": "https://brain/schemas/design-system/v1/foundations.schema.json#/$defs/sourceRef" } },
        "usedBy":     { "type": "array", "items": { "type": "string" } },
        "_gaps":      { "type": "array", "items": { "type": "string" } }
      }
    }
  }
}
```

---

## 6. template.schema.json

```json
{
  "$id": "https://brain/schemas/design-system/v1/template.schema.json",
  "type": "object",
  "required": ["templates"],
  "properties": {
    "templates": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "name", "layer", "slots"],
        "properties": {
          "id":    { "type": "string", "pattern": "^template\\." },
          "name":  { "type": "string" },
          "layer": { "const": "template" },
          "slots": {
            "type": "array",
            "items": {
              "type": "object",
              "required": ["name"],
              "properties": {
                "name":             { "type": "string" },
                "width":            { "type": "string" },
                "height":           { "type": "string" },
                "allowedOrganisms": { "type": "array", "items": { "type": "string" } },
                "required":         { "type": "boolean" }
              }
            }
          },
          "grid": {
            "type": "object",
            "properties": {
              "columns":  { "type": ["integer", "string"] },
              "gutter":   { "type": "string" },
              "maxWidth": { "type": "string" }
            }
          },
          "breakpoints": {
            "type": "array",
            "items": {
              "type": "object",
              "required": ["minWidth", "layout"],
              "properties": {
                "minWidth": { "type": ["integer", "string"] },
                "layout":   { "type": "string" }
              }
            }
          },
          "sourceRefs": { "type": "array", "items": { "$ref": "https://brain/schemas/design-system/v1/foundations.schema.json#/$defs/sourceRef" } },
          "usedBy":     { "type": "array", "items": { "type": "string" } }
        }
      }
    }
  }
}
```

---

## 7. page.schema.json

```json
{
  "$id": "https://brain/schemas/design-system/v1/page.schema.json",
  "type": "object",
  "required": ["pages"],
  "properties": {
    "pages": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "name", "layer", "templateRef", "tree"],
        "properties": {
          "id":          { "type": "string", "pattern": "^page\\." },
          "name":        { "type": "string" },
          "layer":       { "const": "page" },
          "templateRef": { "type": "string" },
          "tree": {
            "type": "array",
            "items": {
              "type": "object",
              "required": ["slot"],
              "properties": {
                "slot":        { "type": "string" },
                "organismRef": { "type": "string" },
                "moleculeRef": { "type": "string" },
                "atomRef":     { "type": "string" },
                "props":       { "type": "object" },
                "children":    { "type": "array" }
              }
            }
          },
          "copy": { "type": "object" },
          "states": {
            "type": "array",
            "items": { "enum": ["default", "empty", "loading", "error", "offline", "disabled"] }
          },
          "routeHints": {
            "type": "object",
            "properties": {
              "path":    { "type": "string" },
              "params":  { "type": "array", "items": { "type": "string" } }
            }
          },
          "sourceRef": { "$ref": "https://brain/schemas/design-system/v1/foundations.schema.json#/$defs/sourceRef" }
        }
      }
    }
  }
}
```

---

## 8. use-case.schema.json

```json
{
  "$id": "https://brain/schemas/design-system/v1/use-case.schema.json",
  "type": "object",
  "required": ["useCases"],
  "properties": {
    "useCases": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "name", "steps"],
        "properties": {
          "id":            { "type": "string", "pattern": "^flow\\." },
          "name":          { "type": "string" },
          "goal":          { "type": "string" },
          "actor":         { "type": "string" },
          "preconditions": { "type": "array", "items": { "type": "string" } },
          "steps": {
            "type": "array",
            "items": {
              "type": "object",
              "required": ["step", "pageRef"],
              "properties": {
                "step":    { "type": "integer", "minimum": 1 },
                "pageRef": { "type": "string" },
                "state":   { "type": "string" },
                "trigger": { "type": "string" },
                "action":  { "type": "string" },
                "data":    { "type": "object" }
              }
            }
          },
          "transitions": {
            "type": "array",
            "items": {
              "type": "object",
              "properties": {
                "from":     { "type": "integer" },
                "to":       { "type": "integer" },
                "on":       { "type": "string" },
                "duration": { "type": "string" },
                "motion":   { "type": "string" }
              }
            }
          },
          "successPath": { "type": "array", "items": { "type": "integer" } },
          "failurePaths": {
            "type": "array",
            "items": {
              "type": "object",
              "properties": {
                "reason": { "type": "string" },
                "steps":  { "type": "array", "items": { "type": "integer" } }
              }
            }
          },
          "sourceRefs": { "type": "array", "items": { "$ref": "https://brain/schemas/design-system/v1/foundations.schema.json#/$defs/sourceRef" } }
        }
      }
    }
  }
}
```

---

## 9. mappings schemas

### token-map.schema.json

```json
{
  "$id": "https://brain/schemas/design-system/v1/token-map.schema.json",
  "type": "object",
  "additionalProperties": {
    "type": "object",
    "required": ["id", "values"],
    "properties": {
      "id":        { "type": "string" },
      "kind":      { "enum": ["color", "typography", "spacing", "radius", "elevation", "motion", "iconography"] },
      "values":    { "type": "object", "additionalProperties": { "type": ["string", "number"] } },
      "role":      { "type": "string" },
      "aliasFor":  { "type": "string" },
      "aliasChain":{ "type": "array", "items": { "type": "string" } },
      "usedBy":    { "type": "array", "items": { "type": "string" } }
    }
  }
}
```

### component-index.schema.json

```json
{
  "$id": "https://brain/schemas/design-system/v1/component-index.schema.json",
  "type": "object",
  "additionalProperties": {
    "type": "object",
    "required": ["id", "layer"],
    "properties": {
      "id":             { "type": "string" },
      "layer":          { "enum": ["primitive", "atom", "molecule", "organism", "template", "page"] },
      "name":           { "type": "string" },
      "sourceRefs":     { "type": "array" },
      "variants":       { "type": "array", "items": { "type": "string" } },
      "states":         { "type": "array", "items": { "type": "string" } },
      "usesTokens":     { "type": "array", "items": { "type": "string" } },
      "usesComponents": { "type": "array", "items": { "type": "string" } }
    }
  }
}
```

### dependency-graph.schema.json

```json
{
  "$id": "https://brain/schemas/design-system/v1/dependency-graph.schema.json",
  "type": "object",
  "required": ["nodes", "edges"],
  "properties": {
    "nodes": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "layer"],
        "properties": {
          "id":    { "type": "string" },
          "layer": { "enum": ["token", "primitive", "atom", "molecule", "organism", "template", "page", "flow"] }
        }
      }
    },
    "edges": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["from", "to", "kind"],
        "properties": {
          "from": { "type": "string" },
          "to":   { "type": "string" },
          "kind": { "enum": ["uses", "composes", "slots", "sequences"] }
        }
      }
    }
  }
}
```

### state-coverage.schema.json

```json
{
  "$id": "https://brain/schemas/design-system/v1/state-coverage.schema.json",
  "type": "object",
  "additionalProperties": {
    "type": "object",
    "required": ["componentId", "variantCoverage"],
    "properties": {
      "componentId":     { "type": "string" },
      "variantCoverage": {
        "type": "object",
        "additionalProperties": {
          "type": "object",
          "required": ["documented", "missing", "inapplicable"],
          "properties": {
            "documented":   { "type": "array", "items": { "type": "string" } },
            "missing":      { "type": "array", "items": { "type": "string" } },
            "inapplicable": { "type": "array", "items": { "type": "string" } }
          }
        }
      },
      "completeness": { "type": "number", "minimum": 0, "maximum": 1 }
    }
  }
}
```

---

## 10. gaps.schema.json

```json
{
  "$id": "https://brain/schemas/design-system/v1/gaps.schema.json",
  "type": "object",
  "required": ["gaps"],
  "properties": {
    "gaps": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "kind", "severity", "message"],
        "properties": {
          "id":       { "type": "string" },
          "kind":     { "enum": ["missing-state", "unresolved-ref", "conflicting-token", "ambiguous-atom", "orphan-component", "anonymous-literal", "degraded-track", "schema-violation"] },
          "severity": { "enum": ["error", "warning", "info"] },
          "target":   { "type": "string", "description": "componentId or tokenId the gap is about" },
          "message":  { "type": "string" },
          "context":  { "type": "object" },
          "sourceRefs": { "type": "array" }
        }
      }
    }
  }
}
```

---

## Validation in the synthesizer

```
for each layer in [foundations, atoms, molecules, organisms, templates, pages, useCases]:
  let schema = load("<output>/schemas/<layer>.schema.json")
  let data   = load("<output>/<layer>.json")
  let result = ajv.validate(schema, data)
  if not result:
    emit gap(kind="schema-violation", severity="warning",
             target=layer, message=result.errors,
             context={...})
```

Validation failures become `gaps.json` entries — never blockers. The spec emits imperfect over missing.

---

## Adding new token kinds

Extend the relevant `$defs` in `foundations.schema.json` and add a new group under `colors` / `typography` / etc. ID prefixes stay `{kind}.` (e.g. `gradient.`, `blur.`, `scroll-snap.`). Downstream tooling reads `token-map.json` — if the id prefix is new, the tooling treats it as an opaque reference.

---

## 11. semantic-tokens.schema.json (v2)

Semantic layer atop concept tokens. See `docs/SEMANTIC-TOKENS.md` for the canonical vocabulary.

```json
{
  "$id": "https://brain/schemas/design-system/v2/semantic-tokens.schema.json",
  "type": "object",
  "required": ["meta", "tokens"],
  "properties": {
    "meta": {
      "type": "object",
      "required": ["themes", "vocabularyVersion", "derivedFrom"],
      "properties": {
        "themes":            { "type": "array", "items": { "type": "string" }, "minItems": 1 },
        "vocabularyVersion": { "type": "string" },
        "derivedFrom":       { "type": "string" }
      }
    },
    "tokens": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "category", "purpose", "mappings", "cssVar"],
        "properties": {
          "id": {
            "type": "string",
            "pattern": "^semantic\\.(surface|text|border|accent|state|spacing|type|radius|elevation|motion|domain)\\."
          },
          "category": {
            "enum": ["surface", "text", "border", "accent", "state", "spacing", "type", "radius", "elevation", "motion", "domain"]
          },
          "purpose": { "type": "string" },
          "mappings": {
            "type": "object",
            "additionalProperties": {
              "type": "object",
              "properties": {
                "conceptRef": { "type": "string" },
                "literal":    { "type": ["string", "number"] }
              }
            }
          },
          "cssVar": { "type": "string", "pattern": "^--" },
          "aliasFor": { "type": "string" },
          "usedBy": { "type": "array", "items": { "type": "string" } }
        }
      }
    },
    "_gaps": { "type": "array", "items": { "type": "string" } }
  }
}
```

**Component tokenRef update**: atom/molecule/organism/template/page tokenRefs SHOULD use `semantic.*` prefix. Concept prefixes (`color.*`, `space.*`, etc.) are still valid but emit a `concept-only-ref` gap.

---

## 12. snapshot-metadata.schema.json (v2)

Index of every PNG snapshot emitted by `scripts/snapshot.mjs`.

```json
{
  "$id": "https://brain/schemas/design-system/v2/snapshot-metadata.schema.json",
  "type": "object",
  "required": ["generatedAt", "themes", "viewport", "snapshots"],
  "properties": {
    "generatedAt": { "type": "string", "format": "date-time" },
    "themes":      { "type": "array", "items": { "type": "string" } },
    "viewport": {
      "type": "object",
      "required": ["width", "height"],
      "properties": {
        "width":  { "type": "integer", "minimum": 1 },
        "height": { "type": "integer", "minimum": 1 }
      }
    },
    "puppeteerVersion": { "type": "string" },
    "totals": {
      "type": "object",
      "properties": {
        "snapshots": { "type": "integer", "minimum": 0 },
        "errors":    { "type": "integer", "minimum": 0 }
      }
    },
    "snapshots": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "file", "layer", "theme", "sourceHtml", "dimensions", "takenAt"],
        "properties": {
          "id":         { "type": "string" },
          "file":       { "type": "string" },
          "layer":      { "enum": ["foundations", "atoms", "molecules", "organisms", "templates", "pages", "use-cases"] },
          "theme":      { "type": "string" },
          "sourceHtml": { "type": "string" },
          "dimensions": {
            "type": "object",
            "required": ["width", "height"],
            "properties": {
              "width":  { "type": "integer", "minimum": 1 },
              "height": { "type": "integer", "minimum": 1 }
            }
          },
          "viewport": {
            "type": "object",
            "properties": {
              "width":  { "type": "integer" },
              "height": { "type": "integer" }
            }
          },
          "takenAt": { "type": "string", "format": "date-time" }
        }
      }
    },
    "errors": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "file":   { "type": "string" },
          "theme":  { "type": "string" },
          "id":     { "type": "string" },
          "reason": { "type": "string" }
        }
      }
    }
  }
}
```

## 13. manifest.schema.json (v2 updates)

v2 adds `cacheKeys`, `runtimeVersions`, `previousRuns`.

```json
{
  "$id": "https://brain/schemas/design-system/v2/manifest.schema.json",
  "allOf": [
    { "$ref": "https://brain/schemas/design-system/v1/manifest.schema.json" },
    {
      "type": "object",
      "properties": {
        "cacheKeys": {
          "type": "object",
          "properties": {
            "foundations":         { "type": "string" },
            "semantic-tokens":     { "type": "string" },
            "atoms":               { "type": "string" },
            "molecules-organisms": { "type": "string" },
            "templates-pages":     { "type": "string" },
            "use-cases":           { "type": "string" },
            "html":                { "type": "string" },
            "snapshots":           { "type": "string" }
          }
        },
        "runtimeVersions": {
          "type": "object",
          "properties": {
            "skill":     { "type": "string" },
            "node":      { "type": "string" },
            "puppeteer": { "type": "string" },
            "chromium":  { "type": "string" }
          }
        },
        "previousRuns": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "at":      { "type": "string", "format": "date-time" },
              "skill":   { "type": "string" },
              "layers":  { "type": "array", "items": { "type": "string" } },
              "cached":  { "type": "array", "items": { "type": "string" } }
            }
          }
        }
      }
    }
  ]
}
```
