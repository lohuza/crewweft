# Crewweft

Status: architecture and delivery plan awaiting approval. Implementation has not started.

Build a Go terminal application for Windows, macOS, and Linux that coordinates installed coding agents through an approved plan, implementation, independent review, and a human decision. Preserve project architecture, decisions, and implementation history in PostgreSQL with pgvector.

The workflow is: gather context, write documents and a plan, obtain approval, implement, run checks, review, resolve disagreements with the human, and save the outcome for future work.

## First release delivery

- A Go executable, `crewweft`, with a terminal interface for Windows, macOS, and Linux.
- Detection and refresh of installed Codex, Claude Code, and Cursor Agent tools, with configurable roles and models.
- Repository detection and isolated task workspaces; one writing task at a time by default.
- Approved documents and plans, configurable steps, independent checks/review, and an inbox for questions and human decisions.
- PostgreSQL with pgvector through Docker or a supplied connection string, preserving searchable project knowledge and source evidence.
- Optional read-only MCP context gathering, with Jira as the first integration example, and a local MCP server exposing Crewweft memory.
- Shared non-secret integration configuration with locally stored credentials and authentication on each new computer.
- Tests, native platform verification, setup instructions, and backup/recovery documentation.

Delivery is complete when a user can finish an approved task through implementation, verification, review, and a human decision, then retrieve its decisions and evidence in a later session. The release must also demonstrate question handling, disagreement escalation, and recovery without duplicate work. Supported provider/platform combinations must be tested and documented.

## Later delivery

Local model runtimes, verified internal/offline operation, shared secrets-manager integrations, hosted memory MCP, and multiple concurrent writing tasks follow in separately approved stages. Commits, pushes, merges, and PR management remain the user's responsibility in the first release.

## Documents

1. [Architecture and requirements](docs/architecture.md): what Crewweft does, how it works, data and credential handling, integrations, and release boundaries.
2. [Implementation and delivery plan](docs/implementation-plan.md): ordered work, affected packages, tests, and acceptance criteria for each stage.

All command examples describe the proposed product; they are not available yet.

No implementation should begin until the architecture and implementation plan are explicitly approved. Material changes to the approved scope require a documented amendment.
