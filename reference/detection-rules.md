# Detection Rules

Playbook for detecting project stacks and instantiating the correct agent templates. Used by SKILL.md step 3 (existing project path).

## A. File Signature Table

Each row: signature file/pattern → detected stack → agent to instantiate (template path) → default gate commands → default model tier suggestion.

| Signature | Stack | Agent template | Default gates | Model tier |
|-----------|-------|----------------|---------------|------------|
| `go.mod` | Go backend | `templates/agents/stack/develop-backend.md` | `go build ./...`, `go test ./...`, `go vet ./...`, `golangci-lint run ./...` | mid |
| `package.json` + `next.config.*` | Next.js | `templates/agents/stack/develop-frontend.md` | `npx tsc --noEmit`, `npm test` or `bun test`, `npm run lint` | mid |
| `package.json` + `nuxt.config.*` | Nuxt | (no shipped template — propose general-coding until template exists) | `npx nuxi typecheck`, `npm test`, `npm run lint` | mid |
| `e2e/` + `playwright.config.*` | Playwright E2E | (extends qa/verifier gates) | `npx playwright test` or project make target | cheap-mid |
| `cypress.config.*` + `cypress/` | Cypress E2E | (extends qa/verifier gates) | `npx cypress run` or project make target | cheap-mid |
| `Dockerfile` | Container | (verifier gate addition) | `docker build -t validate-test .` | — |
| `docker-compose.yml` or `compose.yml` | Container orchestration | (verifier gate addition) | `docker compose up --build` (if applicable) | — |
| `Makefile` present | Build orchestration | — | prefer make targets over raw commands when a target exists for a gate | — |
| `*/servicebus*`, `azure-sdk` deps in go.mod/package.json | Message bus | `templates/agents/stack/develop-service-bus.md` | stack-native test commands | mid |
| `*/kafka*`, `sarama`/`confluent-kafka` deps | Kafka consumer | (no shipped template — propose general-coding until template exists) | stack-native test commands | mid |
| `*/sqs*`, `aws-sdk` deps + consumer pattern | SQS consumer | (no shipped template — propose general-coding until template exists) | stack-native test commands | mid |
| `pytest.ini`/`pyproject.toml` + `pytest`, `manage.py` | Python (Django/Flask/FastAPI) | (no shipped template — propose general-coding until template exists) | `pytest`, `ruff check .` or `flake8` | mid |
| `Cargo.toml` | Rust | (no shipped template — propose general-coding until template exists) | `cargo build`, `cargo test`, `cargo clippy` | mid |
| `Gemfile` + `Rakefile` | Ruby (Rails) | (no shipped template — propose general-coding until template exists) | `bundle exec rspec`, `bundle exec rubocop` | mid |
| `pom.xml`/`build.gradle` | Java (Maven/Gradle) | (no shipped template — propose general-coding until template exists) | `mvn test` or `gradle test`, `mvn checkstyle:check` | mid |
| `*.csproj`/`*.sln` | .NET (C#) | (no shipped template — propose general-coding until template exists) | `dotnet build`, `dotnet test`, `dotnet format --verify-no-changes` | mid |

**Notes:**
- "mid" = mid-tier model (e.g. `bifrost/dashscope/qwen3.7-plus`)
- "cheap-mid" = cheap-to-mid model (e.g. `bifrost/dashscope/qwen3.7-plus` or flash variant)
- "—" = not applicable (verifier gate addition only, no dedicated agent)
- Stacks without shipped templates → propose `general-coding` agent until a dedicated template exists

## B. Detection Procedure

Ordered steps for detecting project stacks:

1. **Glob manifests at repo root + one level deep**
   - `go.mod`, `package.json`, `Cargo.toml`, `pyproject.toml`, `Gemfile`, `pom.xml`, `build.gradle`, `*.csproj`, `*.sln`
   - One level deep: `*/go.mod`, `*/package.json`, etc. (monorepo signal)

2. **Read Makefile/package.json scripts for gate commands**
   - If `Makefile` exists → parse targets (look for `test`, `lint`, `build`, `test-unit`, `test-it`, etc.)
   - If `package.json` exists → parse `scripts` field (look for `test`, `lint`, `build`, `typecheck`, etc.)
   - **Makefile targets take PRIORITY** over raw commands when a target exists for a gate

3. **Scan for service-bus dependencies**
   - Check `go.mod` require lines / `package.json` dependencies for messaging SDKs (`azservicebus`, `kafka-go`, `sqs`, `amqp`)
   - If found → flag service-bus layer → instantiate `develop-service-bus` template
   - Check Makefile for bus-specific test targets (e.g. `test-it-sb` pattern)

4. **Read CI config for authoritative gate commands**
   - `.github/workflows/*.yml` → parse `run:` steps in CI jobs
   - `.gitlab-ci.yml` → parse `script:` sections
   - `.circleci/config.yml`, `azure-pipelines.yml`, `Jenkinsfile` → similar
   - CI config is authoritative — if CI runs `make ci-test`, that's the canonical test gate

5. **Detect monorepo**
   - Multiple manifests in subdirs (e.g. `frontend/package.json` + `backend/go.mod`) → one agent per layer with per-layer dirs
   - Each layer gets its own agent instantiation with layer-specific gates and dirs

6. **Detect E2E / integration test layers**
   - `e2e/`, `tests/e2e/`, `cypress/`, `playwright/` → extends verifier gates, may need dedicated agent if complex

7. **Detect container setup**
   - `Dockerfile`, `docker-compose.yml` → adds verifier gate for build validation

## C. Proposal Table Format

The exact table shape SKILL.md step 3.2 shows. Every cell filled or explicitly marked "ask user".

| Agent | Model | Purpose | Gate commands |
|-------|-------|---------|---------------|
| verifier | cheap-flash | gate runner | `<!-- filled from detection -->` |
| qa | mid-tier | AC verification | — |
| general-coding | mid-tier | universal worker | — |
| `develop-<stack>` | mid-tier | domain worker | `<!-- filled from detection -->` |

**Instructions:**
- **Agent**: agent name (matches template filename without `.md`)
- **Model**: model tier from signature table, or user override
- **Purpose**: one-line description of what this agent does
- **Gate commands**: exact commands this agent runs (from detection procedure, or "ask user" if ambiguous)
- If any cell cannot be filled from detection → mark "ask user" and grill during step 3.3

## D. Ambiguity Rules

Rules for resolving ambiguous detection signals:

1. **Multiple test frameworks detected** (e.g. both Jest and Vitest in `package.json`)
   - → Ask user which is canonical
   - Do NOT guess — wrong test framework = broken gates

2. **No CI config + no Makefile**
   - → Default gates from signature table
   - → Confirm with user before writing (show proposed gates, ask "correct?")

3. **Conflicting signals in same dir** (e.g. `go.mod` + `package.json` in root)
   - → Treat as monorepo signal
   - → Ask user: "Detected Go + Node in same directory. Is this a monorepo with separate layers, or a single project using both?"

4. **Multiple possible agent templates for one stack** (e.g. Next.js + Playwright)
   - → Instantiate both: `develop-frontend` for Next.js, verifier gates extended for Playwright
   - → Ask user if they want a dedicated E2E agent or just verifier gates

5. **Unknown stack** (signature detected but no shipped template)
   - → Propose `general-coding` agent
   - → Ask user: "No dedicated template for <stack>. Use general-coding, or skip dedicated agent?"

6. **Custom build system** (e.g. Bazel, Nx, Turborepo)
   - → Parse build system config for gate commands
   - → Ask user to confirm detected gates

7. **No tests detected**
   - → Ask user: "No test framework detected. Add test gates later, or skip test gates for now?"

## E. Detection Output

After detection, produce:

1. **Detected stacks list** — one line per stack with signature file path
2. **Proposed agent table** — per section C format
3. **Proposed gate table** — per agent, exact commands
4. **Ambiguities** — list of questions for user (per section D)

Example output:

```
## Detected stacks

- Go backend: `backend/go.mod`
- Next.js frontend: `frontend/package.json` + `frontend/next.config.js`
- Playwright E2E: `e2e/playwright.config.ts`

## Proposed agents

| Agent | Model | Purpose | Gate commands |
|-------|-------|---------|---------------|
| verifier | cheap-flash | gate runner | see below |
| qa | mid-tier | AC verification | — |
| general-coding | mid-tier | universal worker | — |
| develop-backend | mid-tier | Go backend worker | `go build ./...`, `make test-unit`, `golangci-lint run ./...` |
| develop-frontend | mid-tier | Next.js frontend worker | `npx tsc --noEmit`, `npm test`, `npm run lint` |

## Gate commands (verifier)

Backend:
- `go build ./...` (workdir: `backend/`)
- `make test-unit` (workdir: `backend/`)
- `golangci-lint run ./...` (workdir: `backend/`)

Frontend:
- `npx tsc --noEmit` (workdir: `frontend/`)
- `npm test` (workdir: `frontend/`)
- `npm run lint` (workdir: `frontend/`)

E2E:
- `npx playwright test` (workdir: `e2e/`)

## Ambiguities

None detected. Proceed? (yes/no)
```
