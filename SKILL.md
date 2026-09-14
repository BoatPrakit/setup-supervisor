---
name: setup-supervisor
description: Bootstrap an opencode Supervisor agent mode in any project — planning protocol, subagent templates, gate validation, and protocol skills. Trigger phrases — "setup supervisor", "configure supervisor mode", "bootstrap supervisor", "create subagents", "init supervisor mode".
disable-model-invocation: true
---

# Setup Supervisor

Bootstrap the Supervisor agent stack for any project:

- **Supervisor protocol** — `AGENTS.md` with planning, delegation, validation, and memory-capture rules
- **Core subagents** — `verifier` (gate runner), `qa` (AC verification), `general-coding` (universal worker)
- **Protocol skills** — `grilling`, `wayfinder`, `tdg` (vendored copies)
- **Stack-specific agents** — domain workers detected from project files (later step)

This is a prompt-driven skill, not a deterministic script. Explore, confirm with the user, then write.

## Process

### 1. Ask install scope

Ask the user where to install:

| Scope | Protocol + agents | Skills |
|-------|-------------------|--------|
| **User** | `~/.config/opencode/agent/` | `~/.agents/skills/` |
| **Project** | `.opencode/agent/` in repo root | `.agents/skills/` in repo root |
| **Both** | user-scope core + project-scope stack agents | both |

Default recommendation: **user-scope** for core agents (shared across projects), **project-scope** for stack-specific agents and skills. Record the choice.

### 2. Detect project state

Determine whether the target is:

- **Empty/new repo** — no source files, no `package.json`/`go.mod`/`Cargo.toml`/etc. → go to step 4 (new project path)
- **Existing codebase** — has source files, dependency manifests, tests → go to step 3 (existing project path)

### 3. Existing project path

1. **Detect stacks** using rules in `reference/detection-rules.md` (later step file). Scan for:
   - Language runtimes (Go, TypeScript, Python, Rust, etc.)
   - Framework signals (Next.js, Django, Rails, etc.)
   - Test frameworks (Jest, Vitest, Go test, pytest, etc.)
   - Build tools (Make, Turborepo, Nx, etc.)
   - E2E tools (Playwright, Cypress, etc.)
   - Container setup (Dockerfile, docker-compose)

2. **Propose agent set + gates** as a table:

   | Agent | Model | Purpose | Gate commands |
   |-------|-------|---------|---------------|
   | verifier | cheap-flash | gate runner | none — receives gates from supervisor |
   | qa | mid-tier | AC verification | none — delegates to workers |
   | general-coding | mid-tier | universal worker | — |
   | `develop-<stack>` | mid-tier | domain worker | `<!-- filled from detection -->` |

3. **Grill user** on gaps and overrides:
   - Missing stack agents they want?
   - Gate commands correct? Any custom Make targets?
   - Model preferences per agent?
   - Commit message conventions (ticket prefix, author name)?

4. **Confirm** before writing. Show final agent list + gate table.

5. **Model assignment.** Run the procedure below to assign a concrete model ID to every agent in the confirmed proposal table.

   **Harvest suggestions (read-only, never print auth keys/tokens — model IDs only):**

   Union of these sources, deduped:

   - **Config files** — `~/.config/opencode/opencode.jsonc` + project `opencode.jsonc` / `.opencode/project.json`: extract provider/model blocks (the keys inside `"providers"` and their `"models"` maps).
   - **Agent frontmatter** — `rg "^model:" ~/.config/opencode/agent/` + `rg "^model:" .opencode/agent/` (if present).

   Collect every distinct `provider/model-id` string. Discard duplicates.

   **Tier tagging by name heuristic:**

   | Pattern in model ID | Tier label |
   |---------------------|-----------|
   | `flash\|lite\|haiku\|mini` | cheap |
   | `plus\|pro\|max\|opus` | strong |
   | anything else | mid |

   **Fallback:** if harvest yields zero models (fresh machine, no agents yet), skip the suggestion column — show tier labels only (`cheap` / `mid` / `strong`) and rely on free-text override.

   **Assignment flow:**

   1. Offer **"accept all suggestions?"** first — if yes, apply harvested suggestion per tier mapping and skip per-agent prompts.
   2. Otherwise, walk the confirmed proposal table row by row. For each agent show:

      | Agent | Tier | Suggested model | Alternatives |
      |-------|------|-----------------|--------------|
      | verifier | cheap | `<harvested cheap match>` | `<other cheap IDs>` |
      | qa | mid | `<harvested mid match>` | `<other mid IDs>` |
      | general-coding | mid | `<harvested mid match>` | `<other mid IDs>` |
      | develop-\<stack\> | mid | `<harvested mid match>` | `<other mid IDs>` |

   3. Per agent: confirm suggestion, pick alternative, or **free-text override** (any `provider/model-id` string, even ones not harvested).
   4. Record final per-agent models — they fill `<!-- MODEL: -->` placeholders at write time (step 5/6).

### 4. New project path

Run the full interview per `reference/new-project-interview.md` (later step file). Collect:

- **Stacks/layers** — what languages, frameworks, runtime
- **Project layout** — directory conventions (monorepo? separate frontend/backend?)
- **Test framework + gate commands** — what to run for build/test/lint
- **Subagent naming** — which domain agents (model assignment deferred to step 3.5 procedure)
- **Git/commit conventions** — ticket prefix format, author name, branch naming

After the interview completes, run the **model assignment procedure from step 3.5** (harvest + tier tag + per-agent confirm/override). On a fresh machine the harvest likely returns empty — that's fine, the user supplies real model IDs via free-text override against the tier labels.

Then proceed to step 5 with collected answers + assigned models.

### 5. Write generic core

Copy the following templates to the target scope (from step 1), adapting placeholders:

| Template source | Target |
|-----------------|--------|
| `templates/protocol/AGENTS.md` | `<scope>/AGENTS.md` |
| `templates/agents/generic/verifier.md` | `<scope>/agents/verifier.md` |
| `templates/agents/generic/qa.md` | `<scope>/agents/qa.md` |
| `templates/agents/generic/general-coding.md` | `<scope>/agents/general-coding.md` |

Placeholder adaptation rules:
- `<!-- GATE: detected-per-project -->` → fill with detected gate commands from step 3/4
- `<!-- GATES: filled from detection-rules at install -->` → fill with per-agent gate lists
- `<!-- MODEL: -->` → fill with the model ID assigned in step 3.5 (per agent, from the final assignment table)
- `<TICKET_PREFIX>` → in installed protocol AGENTS.md commit template; fill from interview section 5 (new project) or step-3 grilling (existing project); keep empty brackets `[]` if no prefix
- `<DEFAULT_AUTHOR>` → in installed protocol AGENTS.md commit template; fill from interview section 5 (new project) or step-3 grilling (existing project); keep empty brackets `[]` if none

### 6. Write project-specific agents

From `templates/agents/stack/*` (later step), write domain-specific worker agents tailored with scouted gates and paths. Each stack agent gets:
- Correct model assignment
- Stack-appropriate gate commands in its prompt
- Project-specific directory conventions

Skip if no stack agents detected.

- **Directory maps**: detect actual dirs from the project. For new/empty trees, write canonical layout marked `<!-- suggested, not detected -->`.
- **Undetected project artifacts** (OpenAPI files, convention docs): write `not detected — add when created` and move on. Do not block.
- **Raw commands**: when a signature-table gate has no Makefile target, use the raw command but flag in the proposal table: `raw command (no Makefile target found)`.
- **Skipped templates**: list stack templates detected-but-not-needed (e.g. `develop-service-bus: available, no bus deps detected — skipped`) in the post-install summary.

### 7. Install protocol skills

Install = copy `templates/skills/<name>/` to the skills scope chosen in step 1. Copy vendored `templates/skills/*` (grilling, wayfinder, tdg — later step):

- If skill directory does NOT exist → copy from template
- If skill directory exists → compare `template-version` markers (installed copy vs template copy; for vendored skills the marker is an HTML comment `<!-- template-version: N -->` after the frontmatter):
  - Template version > installed version → report drift, ask user before overwriting
  - Template version <= installed version → skip, report as up-to-date

- **Note**: tdg references a project `TDG.md` (tech-stack doc). Not shipped with this skill — if absent, tell user to create one (or skip tdg usage until it exists).

### 8. Memory plugin opt-in

Summarize the memory plugin design from `reference/memory-plugin.md` (later step file):

- What it does: persistent cross-session lessons (do/dont/context)
- How it works: supervisor records lessons after task completion, injects relevant ones into subagent prompts
- Storage: per-project bucket + global bucket, 20-lesson cap per bucket

Ask user: **Install memory plugin? (yes/no)**

Only install on explicit consent. If yes, copy memory plugin files to the appropriate scope.

### 9. Config snippets

Print configuration snippets from `reference/config-snippets.md` for settings the skill must NOT auto-edit:

- User's global `~/.config/opencode/opencode.jsonc`:
  - `subagent_depth` — nesting depth for subagent spawning
  - Permission rules — file access patterns
- Any other global settings that affect supervisor behavior

Present as copy-paste blocks. Explain each setting. Let user decide whether to apply.

### 10. Post-setup checklist

Report to user:

```
## Setup complete

### Files created
- <path>/AGENTS.md
- <path>/agents/verifier.md
- <path>/agents/qa.md
- <path>/agents/general-coding.md
- <path>/agents/<stack-specific>.md (if any)
- <path>/skills/grilling/SKILL.md (if installed)
- <path>/skills/wayfinder/SKILL.md (if installed)
- <path>/skills/tdg/SKILL.md (if installed)

### Action required
- Restart opencode to discover new agents
- Verify: agents should appear in the agent picker

### Optional next steps
- Review AGENTS.md and adjust gate commands if needed
- Add stack-specific agents as your project evolves
- Run this skill again to fill gaps (never overwrites)
```

### 11. Re-run behavior

When this skill runs on a project that already has supervisor files:

- **Fill gaps only** — never overwrite existing files
- **Version drift report** — compare `template-version` markers in installed files vs templates. This applies to BOTH agent templates (where `template-version` is a frontmatter field) AND vendored protocol skills (where `template-version` is an HTML comment marker after the frontmatter):
  - If installed version < template version → report available upgrades, ask before updating
  - If installed version >= template version → skip, report as current
- **New agents** — if detection finds stacks not yet covered, propose new stack agents
- **Missing skills** — if protocol skills are missing, offer to install them
- **Version markers differ by design**: agent/protocol templates carry `template-version` in frontmatter; vendored skills carry it as an HTML comment below frontmatter (opencode parses skill frontmatter strictly).

## Hard rules

- **Never edit files the skill didn't create.** If the user has custom agents or protocol files, leave them alone.
- **Never `git add -A` anywhere.** Stage only specific files the skill created.
- **Never read `.env` files.** If environment variables are needed, source them without exposing values.
- **Never modify the source templates.** Templates in this skill's directory are read-only references.
