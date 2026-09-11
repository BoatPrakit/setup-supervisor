# Memory Plugin

Two parts: (A) design explainer for the user, (B) opt-in install steps.

---

## Part A — Design explainer

Present this to the user before asking "Install memory plugin? (yes/no)".

### Purpose

Persistent cross-session lessons. The supervisor records insights after task completion (what worked, what failed, domain knowledge). These lessons survive restarts and are injected into future subagent prompts — so the agent gets smarter over time instead of repeating mistakes.

**Lesson types:**
- **do** — what to do (e.g. "Normalize nil slices before pq.Array")
- **dont** — what not to do (e.g. "Never assert toast visibility in e2e")
- **context** — general knowledge (e.g. "Text token scale: text-lg=16px not text-md")

### Architecture

An opencode plugin providing three capabilities:

1. **`memory` tool** — actions: `record`, `update`, `list`, `search`, `show`, `demote`, `promote`. Supervisor calls this to manage lessons.
2. **System-prompt injection hook** — on every chat, the plugin surfaces active lessons (project bucket + global bucket) into the system prompt. Agents see relevant context without querying.
3. **Permission lockdown** — `memory` tool allowed for supervisor only, denied for all worker agents. Workers cannot record lessons (only supervisor curates).

### Storage

**File layout:**
```
memories/
├── <project-bucket>/
│   ├── normalize-nil-slices.md
│   ├── never-assert-toast.md
│   └── ...
└── global/
    ├── text-token-scale.md
    └── ...
```

Each lesson is a markdown file with frontmatter:
```markdown
---
title: Normalize nil slices in Go before pq.Array
type: do
created: 2026-09-11T10:00:00Z
updated: 2026-09-11T10:00:00Z
active: true
---
Never COALESCE array params. Normalize nil slices to empty slices in Go code before passing to pq.Array.
```

**Constraints:**
- 20 active lessons per bucket (project or global). Hard cap.
- At capacity, supervisor must evict one lesson (demote it) before recording a new one. Eviction = agent-curated choice (must name target), not automatic LRU.
- Demote = flip `active: false` in frontmatter. File never deleted.
- Promote = flip `active: true` (respects 20-lesson cap).
- Searchable: plugin scans title + body text, returns matching lessons.

**Why markdown files, not a database:** Derived from the Agent-skills pattern — knowledge as versioned, human-readable files the agent treats as its instruction set. Files are grep-able, diff-able, git-trackable. No vendor lock-in. No migration scripts. If the plugin breaks, lessons survive as plain text.

---

## Part B — Opt-in install steps

Run ONLY on explicit user consent ("yes" to "Install memory plugin?").

### Step 1: Copy plugin file

Copy `memory.ts` to the appropriate scope:
- **User scope:** `~/.config/opencode/plugins/memory.ts`
- **Project scope:** `.opencode/plugins/memory.ts` (create `plugins/` dir if missing)

**Note:** Plugin API requires opencode restart to load. After copying, restart opencode.

**Honest limitation:** The plugin implementation (`memory.ts`) must be sourced from the supervisor repo/distribution. This skill references the design; the file ships separately unless bundled. If the file is not available in this skill's templates, inform the user and provide the source location.

### Step 2: Add permission rules

In `~/.config/opencode/opencode.jsonc` (user scope) or `.opencode/project.json` (project scope), add permission rules:

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

**Rule:** Agent-level permissions (inside `agent.<name>.permission`), NOT global rules. Supervisor gets `allow`, all workers get `deny`.

### Step 3: Create memories directory skeleton

```bash
mkdir -p memories/global
touch memories/global/map.md
```

**Note:** Project buckets are created automatically by the plugin when the first lesson is recorded. Only `global/` needs manual creation (optional — plugin creates it on first use too).

### Step 4: Restart and verify

1. Restart opencode
2. Verify: `memory` tool appears in supervisor's tool list
3. Verify: `memory` tool does NOT appear in worker agents' tool lists (check agent picker → agent details)

### Step 5: Honest limitation note

Inform the user:

> The memory plugin is a reference implementation. The plugin file (`memory.ts`) ships with the supervisor distribution. If you're installing from a different source, you may need to obtain the plugin file separately. The design is documented; the implementation must be sourced from the official distribution.

---

## Summary for user

**What you get:**
- Persistent lessons across sessions
- Supervisor curates knowledge base
- Workers see relevant context in prompts
- 20-lesson cap per bucket (project + global)
- Markdown files (human-readable, git-trackable)

**What it costs:**
- Plugin file + permission config
- opencode restart after install
- Supervisor discipline (must record lessons consistently)

**Install? (yes/no)**
