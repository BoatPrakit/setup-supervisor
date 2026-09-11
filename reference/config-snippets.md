# Config Snippets

Copy-paste blocks for settings the skill must NOT auto-edit (SKILL.md step 9). These live in files the user owns — the skill only prints snippets and explains them.

---

## 1. Global config: `~/.config/opencode/opencode.jsonc`

### `subagent_depth`

Controls how deep subagent nesting can go. Supervisor mode requires depth ≥ 2 (supervisor → subagent → nested task).

```jsonc
{
  "subagent_depth": 2
}
```

**Where in JSON:** Top-level key. Add alongside `model`, `provider`, etc.

**Why 2:** Depth 1 = supervisor can spawn subagents but they cannot spawn their own tasks. Depth 2 = subagents can delegate sub-subtasks (useful for complex workflows like explore → scout → report chains). Depth 3+ rarely needed and increases cost.

### Permission rules (generic example)

Supervisor needs `memory: allow`. Worker agents need `memory: deny`. These are per-agent rules inside the `agent` block:

```jsonc
{
  "agent": {
    "supervisor": {
      "permission": {
        "memory": "allow"
      }
    },
    "general-coding": {
      "permission": {
        "memory": "deny"
      }
    },
    "verifier": {
      "permission": {
        "memory": "deny"
      }
    },
    "qa": {
      "permission": {
        "memory": "deny"
      }
    }
  }
}
```

**Where in JSON:** Inside `agent.<agent-name>.permission`. Each agent gets its own permission block.

**Why per-agent:** Memory tool must be supervisor-only. If a worker agent can record lessons, it pollutes the knowledge base with unvetted insights. Supervisor curates; workers execute.

---

## 2. Project config: `.opencode/project.json`

For project-scoped supervisor setup, the same patterns apply but in the project's `.opencode/project.json`:

### Per-agent model assignment

```json
{
  "agent": {
    "develop-backend": {
      "model": "provider/model-name"
    },
    "develop-frontend": {
      "model": "provider/model-name"
    }
  }
}
```

**Where in JSON:** Top-level `agent` key, then per-agent model override.

**Why:** Different agents can use different models. Domain workers often use mid-tier models; supervisor uses strong model for planning. Cost optimization.

### Project permission rules

```json
{
  "permission": {
    "external_directory": {
      "./scripts": "allow",
      "./tmp": "allow"
    }
  }
}
```

**Where in JSON:** Top-level `permission` key.

**Why:** Project-specific directory access. If agents need to write to directories outside the project root (e.g. shared scripts), allow them here.

---

## 3. Important note

**This skill writes new files only.** It does NOT edit existing config files. The snippets above must be merged into existing config by the user.

**Merge hints:**
- `subagent_depth` → add as top-level key if missing, update value if present
- `agent.<name>.permission` → add `permission` block inside existing agent definition, or create new agent block
- `permission.external_directory` → append to existing `external_directory` map if present
- JSON syntax: watch for trailing commas (jsonc allows, strict json does not)

**After editing:** Restart opencode for changes to take effect.
