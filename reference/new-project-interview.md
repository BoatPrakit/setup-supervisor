# New Project Interview

Conducted when SKILL.md step 4 detects an empty/new repo (no source files, no dependency manifests). Collects all inputs needed to instantiate the supervisor stack.

## Interview rules

1. **One question at a time.** Never dump all 5 sections at once. Ask section 1, wait for answer, proceed to section 2, etc.
2. **Recommended default offered with every question.** State the recommended answer, explain why, let user accept/override.
3. **Each answer feeds a specific artifact.** Tell the user which file/placeholder the answer populates — transparency builds trust.
4. **No assumptions.** If user says "I don't know" or "whatever you recommend", use the recommended default and note it in the summary.
5. **Final summary before writing.** After all 5 sections, present an answers summary table. Ask for confirmation before any files are written.

---

## Section 1: Stacks and layers

**Question:** What languages, frameworks, and runtimes does this project use? Is it a monorepo with multiple layers (e.g. Go API + Next.js frontend), a single app, or something else?

**Recommended default:** If the user is unsure, suggest they describe what they're building and you'll infer the stacks.

**Why it matters:** Determines which stack-specific agent templates to instantiate (e.g. `develop-backend`, `develop-frontend`). If no specific stacks detected or user wants minimal setup, only the core agents (verifier, qa, general-coding) are created — no domain workers.

**Feeds into:** Agent file generation (step 6), gate command detection.

---

## Section 2: Project layout

**Question:** What directory conventions do you want? For a monorepo, typical layout is `backend/` + `frontend/` + `e2e/`. For a single app, everything lives in `src/`. What fits your project?

**Recommended default:** Monorepo with `backend/`, `frontend/`, and optional `e2e/` directories — most flexible, scales well.

**Why it matters:** Directory names are injected as `<!-- DIRS: -->` placeholders into agent prompts. Agents use these to scope their work — a backend agent should not touch frontend files. Wrong directory conventions = agents working in wrong places.

**Feeds into:** `<!-- DIRS: -->` placeholders in all agent templates.

---

## Section 3: Test framework and gate commands

**Question:** What build/test/lint commands does each layer use? Do you plan a Makefile to wrap them, or will you use raw commands? What CI provider (if any)?

Sub-questions (ask only if not obvious from section 1):
- Backend: `go build`, `go test`, `golangci-lint`? Or different?
- Frontend: `tsc --noEmit`, `jest`/`vitest`, `eslint`? Or different?
- E2E: Playwright? Cypress? None yet?
- Makefile targets planned? (e.g. `make test`, `make lint`)

**Recommended default:** Makefile with standard targets (`build`, `test`, `lint`) per layer. CI-agnostic (commands work locally and in any CI).

**Why it matters:** Gate commands are embedded in agent prompts (verifier runs them, workers must pass them before committing). Wrong gates = false confidence or blocked workflows.

**Feeds into:** `<!-- GATE: -->` and `<!-- GATES: -->` placeholders in verifier + agent templates.

---

## Section 4: Subagent naming

**Question:** What should domain-specific agents be named? Convention is `develop-<domain>` (e.g. `develop-backend`, `develop-frontend`).

**Recommended default:**
| Agent | Purpose |
|-------|---------|
| supervisor | planning, delegation, validation |
| verifier | gate runner |
| qa | AC verification |
| general-coding | universal worker |
| develop-\<domain\> | domain worker (one per detected stack) |

**Why it matters:** Agent names become file names in `.opencode/agent/` or `~/.config/opencode/agent/`. Names should reflect the domain they own.

**Feeds into:** Agent filenames and frontmatter `name` field.

**Model assignment is deferred.** After the interview completes, the SKILL.md step 3.5 harvest flow runs: it scans the user's existing opencode config + agent frontmatter for model IDs, tags them by tier heuristic (cheap/mid/strong), and walks the user through per-agent confirm/override. On a fresh machine the harvest returns empty — the user supplies real model IDs via free-text override against the tier labels.

---

## Section 5: Git and commit conventions

**Question:** What commit message format do you use? Specifically:
- **Ticket prefix** — e.g. `PROJ-123`, `TASK-456`, or none?
- **Author name** — used in commit template (e.g. your name, team name, or alias)?
- **Branch naming** — e.g. `feature/PROJ-123-description`, `dev/author/description`?

**Recommended default:**
- Ticket prefix: none (can add later)
- Author name: ask the user directly
- Branch naming: `feature/<description>` or `fix/<description>`

**Why it matters:** The supervisor protocol enforces a commit template: `<TICKET_PREFIX> [author] <type>: <description>` when a ticket prefix is configured, or `[author] <type>: <description>` when no ticket prefix exists (omit the ticket segment entirely — never empty brackets). Every subagent commit must follow this. Wrong conventions = inconsistent git history.

**Feeds into:** Supervisor protocol (`AGENTS.md`) commit template, delegation prompts.

---

## Answers summary

After all 5 sections, present this table before writing any files:

| # | Section | Answer | Feeds into |
|---|---------|--------|------------|
| 1 | Stacks/layers | `<answer>` | Agent templates to instantiate |
| 2 | Project layout | `<answer>` | `<!-- DIRS: -->` placeholders |
| 3 | Test framework + gates | `<answer>` | `<!-- GATE: -->` placeholders |
| 4 | Subagent naming | `<answer>` | Agent filenames + frontmatter `name`; model IDs filled later via SKILL.md step 3.5 harvest flow |
| 5 | Git/commit conventions | `<answer>` | Commit template in AGENTS.md |

**Ask:** "Proceed with these answers? (yes/no/changes needed)"

Only after explicit confirmation, continue to SKILL.md step 5 (write generic core).
