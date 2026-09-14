---
name: tdg
description: Test-Driven Generation uses TDD (Test-Driven Development) techniques to generate tests and code in Red-Green-Refactor loops. Key words to detect are tdg, TDG.
---
<!-- template-version: 1 -->

# Test-Driven Generation

## Instructions

Read TDG.md to understand the project's technology stack.
IF TDG.md does not exist, THEN tell user to create one with `/tdg:init` command AND stop.
IF TDG.md is found, THEN read it and identify:
- what is the testing framework
- how to build the project
- how to run a single unit test
- how to run a whole test
- how to run test coverage

## Identify Issue Number for Traceability
Before starting TDG workflow:
1. Check if user mentioned an issue number (e.g., PROJ-1234, #42)
2. Check current branch name for issue reference (e.g., feature/PROJ-1234-add-sort, fix/PROJ-1234)
3. Ask user for issue number if not found: "Which issue are you working on? (e.g., PROJ-1234)"
4. Default prefix is `PROJ`. If user provides only a number (e.g., "1234"), prepend the default prefix to form `PROJ-1234`.
5. Store the full issue key (e.g., `PROJ-1234`) to include in ALL commit messages for traceability

## Steps for Specification (CRITICAL)
- Use this section ONLY IF phase="unknown" or phase="refactor" 
- Before writing any code, you must:
  1) Run test coverage and record the current coverage percentage.
  2) Draft the code in the chat first. DO NOT start writing code.
  3) Write tests for the draft, ensuring at least 1 happy path and N negative tests.
- Work in small increments, focusing on ONE test case at a time.
- Complete one test at a time, the rest cases must be blank or `skip` them first.
- Run the tests to identify what is failing, then make the necessary changes to pass the tests.
- Commit code in GIT using the following comment pattern (commit it together - spec and implementation in one commit):
  - Use `feat` for new features: `"PROJ-1234 [AUTHOR] feat: add user authentication"`
  - Use `fix` for bug fixes: `"PROJ-1234 [AUTHOR] fix: resolve login validation issue"`

## Steps to Refactor
- Use this section ONLY IF phase="green"
- Refactor and optimize the code following best practices.
- Use interfaces extensively to ensure testability.
- Commit the refactored code using:
  `"PROJ-1234 [AUTHOR] refactor: <message>"`
  Example: `"PROJ-1234 [AUTHOR] refactor: extract authentication logic to service"`
  For minor adjustments, use:
  `"PROJ-1234 [AUTHOR] refactor: chore: <message>"`
  Example: `"PROJ-1234 [AUTHOR] refactor: chore: rename variables for clarity"`

## How to commit
- When committing using Git, DO NOT use `git -a` or `git add .`. Commit only files you have just edited.
- Commit message format: `<TICKET_PREFIX> [AUTHOR] <type>: <description>` (with ticket) or `[AUTHOR] <type>: <description>` (no ticket prefix — omit ticket segment entirely)
  - Types: `feat` (new features), `fix` (bug fixes), `refactor`, `refactor: chore`, `docs` (documentation)
  - Examples (with ticket): 
    - `PROJ-1234 [AUTHOR] feat: add user authentication`
    - `PROJ-1234 [AUTHOR] fix: resolve login validation issue`
    - `PROJ-1234 [AUTHOR] docs: add API documentation for auth endpoint`
  - Examples (no ticket):
    - `[AUTHOR] feat: add user authentication`
    - `[AUTHOR] fix: resolve login validation issue`
- IF a ticket prefix is configured, include it. IF no ticket prefix exists, omit the ticket segment entirely (never empty brackets).
- IF user provides only a number, prepend default prefix `PROJ-` to form the full key (e.g., `PROJ-1234`).
- IF user does not have issue for the commit and a ticket prefix is required, THEN help them create by reverse engineering what we're doing as a precise issue description with:
  * Clear title summarizing the feature/fix
  * Acceptance criteria based on tests being written
  * Technical context from the implementation
- Offer to create the issue using GitHub CLI (`gh issue create`) or GitLab CLI (`glab issue create`) and retrieve the issue number for the commit.

## Atomic Commits
- Break down commits into small, logical units that represent single concerns
- Each commit should be self-contained and able to be reverted independently
- Group files that change together logically:
  - Handler + storage + tests + docs for a feature go together
  - Types/interfaces in one commit
  - API client/hooks in one commit
  - UI components in one commit
- Apply common sense: if files are modified together in normal workflow, commit them together
- DO NOT mix unrelated changes in a single commit
- DO NOT commit generated files unless they are intentional changes
- Restore auto-generated files modified by pre-commit hooks to keep working directory clean

## Example Claude TODOs
You must use Todos pattern, like the following example.
☐ Identify issue number (check user message, branch name, or ask; default prefix: PROJ)
☐ Run test coverage to establish baseline
☐ Draft sort library specification
☐ Write test specification
☐ Run the SINGLE test spec and expect the failing test
☐ Implement code to pass tests
☐ Run the SINGLE test spec and expect the passed test
☐ Commit code with "PROJ-<n> [AUTHOR] feat:" or "PROJ-<n> [AUTHOR] fix:" prefix (commit spec and implementation together)
☐ Refactor and optimize (REFACTOR phase)
☐ Commit the refactored code with "PROJ-<n> [AUTHOR] refactor:" prefix

## Closing message
At the end of each TDD cycle, ask something like:

"Would you like me to continue with the next test case using TDG, or
  would you prefer to refactor anything using TDG first?"

Mention "use tdg" or "using tdg" to allow activating this skill.
