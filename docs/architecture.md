# Crewweft architecture

Status: draft for approval. Prepared for Lasha Abazashvili on 2026-11-21. The project name is Crewweft and the executable name is `crewweft`. Source hosting will be on GitHub; no domain is required.

## 1 Purpose and confirmed requirements

Provide one terminal application that coordinates Codex, Anthropic Claude Code, and Cursor Agent on local projects. The application owns workflow state and evidence; agents perform planning, implementation, and review within that workflow.

Confirmed requirements:

- Implement the application in Go and support native Windows, macOS, and Linux.
- Keep the first release entirely in the terminal, including setup and human decisions.
- Require a requirements/architecture document and an implementation plan before implementation. Default to human approval of the initial plan and final result.
- Allow one large implementation step or smaller steps, with configurable intermediate approval checkpoints.
- Give implementation and review agents a persistent communication channel. Escalate explicit disagreements to the human.
- Let any agent ask for missing information. Save questions and answers and resume the relevant work after a response.
- Detect installed agents and provide refresh. Installation, authentication, and compatibility are distinct states.
- Detect the current repository, choose its configured starting revision, and create a task worktree or independent clone when appropriate.
- Store project knowledge and workflow history in PostgreSQL with pgvector. Offer Docker setup and a user-supplied connection string.
- Provide configurable providers, models, execution rules, and workflow behavior.
- Plan a memory MCP interface for external AI clients.
- Gather context from users, repository documents, and configurable MCP integrations such as Jira. Keep sources and refresh history attached to that context.
- Design for later local model support so the complete workflow can eventually operate within a machine or internal network.
- Leave commits, pushes, merges, and PR management to the human in the first release.

## 2 Proposed release boundaries

These boundaries are proposals for approval, not additional confirmed requirements.

The first release is a single-user application with one local supervisor per OS user. It can register multiple projects and keep queued runs, while allowing one writing task at a time by default. A task may have sequential planner, implementer, and reviewer sessions using different providers. PostgreSQL may be local or remote; agent processes and repository files stay on the local machine.

Include all three provider adapters, capability-aware discovery, terminal setup, durable questions/reviews, worktrees and explicit clone mode, full-text and optional configured semantic search, and a local read-only memory MCP server. Add an optional MCP client for selected read-only context gathering, with Jira as the first integration acceptance example. Keyword search remains available if embeddings are not configured or temporarily unavailable. Semantic retrieval is part of the release once an embedding provider is configured.

Defer a browser/desktop interface, multiple human accounts, coordinated execution across machines, automatic integration of independently edited branches, remote authenticated MCP hosting, external memory writes, arbitrary agent plugins, and embedding model installation. Multiple concurrent writing tasks can follow after ownership and recovery behavior are proven.

Local generation models and a verified internal-only execution profile are later stages. Preserve their runtime interfaces now; do not describe the initial cloud-CLI release as fully internal. See [Integrations and local models](#14-integrations-and-local-models) for the proposed boundaries and acceptance criteria.

Native application support does not imply that every version of every provider CLI works on every platform. Unsupported combinations must be visible and blocked for the affected operation, with other providers still usable.

## 3 User journey and terminal interface

### First launch

1. Open `crewweft` in a project folder or open the project picker if outside a repository.
2. Choose local Docker PostgreSQL or an existing PostgreSQL connection. Docker is optional when a connection string is supplied.
3. Validate connectivity, TLS configuration, database/schema permissions, pgvector, and schema version. Show proposed provisioning/migration actions before applying them.
4. Discover installed providers. Show resolved executable, version, authentication status where observable, supported capabilities, and last scan time.
5. Select provider/model choices for planner, implementer, and reviewer. Separate roles can use the same provider, but review uses a fresh session.
6. Register the project and its base-branch/workspace/check configuration.
7. Select an embedding endpoint and model, or explicitly start with keyword search.
8. Discuss requirements and produce documents and a plan. No implementation launches until the required approval is recorded.

Setup/help/discovery can run without a working database. Workflow execution cannot: a durable approval and run record must exist before launching an implementation process. Database failure must not trigger a silent fallback to a different database.

### Main views

| View | User actions |
| --- | --- |
| Conversation | Describe work, discuss architecture, refine a plan, ask about progress |
| Plan | Read document revisions, inspect steps and acceptance criteria, approve or request changes |
| Activity | Follow structured agent events, logs, changes, and check results; pause or cancel a task |
| Inbox | Answer questions, resolve disagreements, handle permission requests, decide final acceptance |
| Agents | Inspect installations and capabilities, refresh discovery, change future-run role defaults |
| Integrations | Configure MCP sources, select allowed read tools, inspect connection status, refresh selected context |
| Projects | Inspect repository identity, base revision, workspace policy, and configured checks |
| Memory | Search architecture, decisions, prior work, and source evidence |
| Settings | Inspect effective configuration and the origin of each value |

All essential actions must be keyboard accessible. Support terminal resize, narrow layouts, plain/no-color output, and a documented help screen. Streaming logs must not steal focus from an unanswered question. Agent output is rendered as text with unsafe terminal control sequences removed.

### Proposed command surface

The TUI and commands use the same application operations. Commands are useful for recovery and automation, but all essential first-release flows remain available in the TUI.

```text
crewweft                         open the terminal interface
crewweft setup                   configure storage, agents, and defaults
crewweft doctor                  report dependencies, storage, and compatibility
crewweft agents list             show provider observations
crewweft agents refresh          rescan provider executables and safe status probes
crewweft integrations list       show configured MCP sources and connection state
crewweft integrations refresh    refresh connection and tool metadata, without importing content
crewweft context refresh <id>    re-read a selected source and show changes
crewweft projects list           show registered projects
crewweft project inspect         inspect the repository at the current location
crewweft runs list               show current and previous work
crewweft run inspect <id>        inspect plan, questions, evidence, and status
crewweft run pause <id>          request a pause at a safe boundary
crewweft run cancel <id>         stop owned processes and retain work
crewweft memory search <query>   retrieve project-scoped knowledge
crewweft memory reindex          rebuild derived search data
crewweft mcp serve               expose local read-only memory tools over stdio
crewweft supervisor status      inspect the local supervisor
```

Commands that submit decisions must accept structured input and identify the exact document/run revision. Do not implement a generic command that lets an agent grant itself approval.

## 4 Workflow and approval rules

### Required planning artifacts

Each task has a versioned requirements/architecture document and an implementation plan. They may be sections of one document or separate linked documents. Required content is the problem, scope, exclusions, architecture decisions, constraints, acceptance criteria, steps, and verification commands/strategy. A step identifies dependencies, intended file areas, and what proves it is done.

Approval binds the exact document and plan revisions, base repository revision, and effective execution policy. A material change to requirements, base revision, scope, or execution policy invalidates dependent approval and returns to planning. Updating progress or adding test evidence does not change the approved specification.

### Run progression

```text
DRAFT -> PLANNING -> AWAITING_PLAN_APPROVAL -> READY
READY -> IMPLEMENTING -> VERIFYING -> REVIEWING
REVIEWING -> IMPLEMENTING                     accepted correction
REVIEWING -> AWAITING_HUMAN                    explicit disagreement
REVIEWING -> READY                            next approved step
REVIEWING -> FINAL_REVIEW                     all steps complete
FINAL_REVIEW -> IMPLEMENTING                  remaining approved-scope correction
FINAL_REVIEW -> AWAITING_FINAL_APPROVAL -> ACCEPTED
```

Any active phase can enter `AWAITING_HUMAN` for a question, `BLOCKED` for unavailable infrastructure, `PAUSED`, `FAILED`, or `CANCELLED`. Store a reason and the precise continuation rather than guessing which phase resumes. An answer may require replanning rather than immediate continuation. A final human rejection can request changes or close the run as `REJECTED`.

The supervisor checks transition preconditions in code. Agent text such as "done" is evidence to inspect, not permission to advance. Check exit status, result validity, unresolved findings, approved scope, and the exact reviewed artifact version.

### Human decisions

Default behavior is approval before implementation and before final acceptance. Per-step approval is optional. A configured final auto-accept policy can replace the final human approval only after the same required checks and reviews pass. Creating a plan remains mandatory; the initial plan approval remains human-controlled in the first release.

An explicit disagreement goes directly to the human by default. Present both arguments, code locations, requirements, and evidence. The human can require the fix, dismiss the finding with a reason, amend the plan, or cancel. A human-authorized exception is recorded as an exception, not falsely labeled as reviewer agreement or passing verification.

Questions include the requesting role, question, why it matters, suggested choices where appropriate, free-text input, affected step, and blocking scope. Blocking questions wait indefinitely until answered or cancelled. No timeout or default selection counts as an answer. Unrelated read-only activity may continue.

Agent permission requests remain distinct from product questions. Forward them only through a supported provider mechanism; otherwise stop with a specific action for the user. Never silently disable provider permissions to keep a run moving.

## 5 Implementation and review communication

The supervisor stores typed messages and routes them to the correct session. No shared mutable chat file is the authority for workflow transitions.

| Message | Required information |
| --- | --- |
| Question | Request ID, task/step, role, question, context, blocking scope |
| Finding | Finding ID, severity, requirement, location, explanation, suggested correction, artifact digest |
| Response | Finding ID, accepted/disputed disposition, rationale, fix/evidence references |
| Review result | Artifact digest, open findings, disposition, limitations |
| Decision | Human identity or configured policy, target revision, choice, rationale, timestamp |
| Handoff | Approved plan revision, completed work, remaining work, evidence, unresolved items |

Only the supervisor or authenticated human UI can create an authoritative decision record. Provider-supplied identities and role labels are untrusted. Bind every message to the run/session actually producing it.

The reviewer examines a stable snapshot of the changes, the approved plan, and check evidence. Implementation pauses during review. The reviewer uses a separate session and provider-enforced read-only permissions where available. Do not claim a prompt alone enforces read-only behavior. If the adapter cannot provide the required restriction, show that capability gap and block that review mode.

Accepted corrections return to the implementer. An implementer response marked disputed creates a human inbox item immediately. Re-run affected checks and review the new snapshot after corrections. Changed source files invalidate older approval even if the agent claims the change is trivial.

Use a configurable bounded repair loop, proposed default three correction cycles per step and three for the final whole-task review. Repeated equivalent failures or unchanged patches stop earlier and request human input. A final review checks the whole task against the complete plan, including interactions between steps.

## 6 Go application architecture

Use one Go codebase and executable with TUI, supervisor, maintenance, and MCP subcommands.

```text
Terminal UI / CLI
       |
Local authenticated API and event stream
       |
Supervisor
  |-- workflow and approval rules
  |-- question and review inbox
  |-- provider adapters and process manager
  |-- MCP client and selected context gathering
  |-- repository/workspace manager
  |-- check runner and artifact snapshots
  |-- memory and indexing jobs
       |
PostgreSQL + pgvector / local artifact files

Local external AI client -> MCP stdio -> same memory application service
```

Proposed libraries are Bubble Tea for the TUI, pgx for PostgreSQL, and the official Go MCP SDK. Bubble Tea provides a Go terminal UI framework; pgx accepts PostgreSQL connection URLs and offers pooling; the MCP project maintains a Go SDK. Pin compatible versions during the first implementation stage. [Bubble Tea](https://github.com/charmbracelet/bubbletea), [pgx](https://github.com/jackc/pgx), [Go MCP SDK](https://github.com/modelcontextprotocol/go-sdk).

Keep business rules independent of terminal rendering and provider command details. Start with standard Go testing and explicit SQL migrations. Exact Go/library versions are chosen together after checking supported releases; the installed Go version is not automatically the project's required version.

An agent runtime performs a complete tool-using turn; a model client only performs inference. Keep these interfaces distinct. Existing CLI adapters implement the runtime boundary. A later local runtime will combine an inference client with Crewweft-controlled tools, questions, and permission handling. Embeddings use their own interface and can be local independently of the coding runtime.

### Supervisor and local transport

The terminal launches or connects to a per-user background supervisor. Closing the UI detaches; explicit shutdown is separate. On Windows, background startup must not open an extra visible console. A machine-local lock prevents duplicate supervisors.

Use loopback HTTP with a per-user secret stored with restricted file permissions, plus an event stream with durable sequence IDs. Bind only to loopback, validate clients/origins, and avoid exposing tokens in arguments or logs. The protocol is local; using a remote database does not expose the supervisor remotely.

Worker tools receive separate run-scoped capabilities that cannot submit human decisions; never pass the UI credential into a worker environment. These controls prevent accidental/protocol-level self-approval. A provider running unrestricted as the same OS user can potentially access that user's files, so strict isolation additionally depends on provider/OS sandboxing. Do not advertise the local API credential alone as a sandbox against arbitrary host code.

PostgreSQL stores current state and append-only workflow events in the same transaction. Event delivery is at least once, with request IDs and deduplication. Reconnecting clients fetch a current snapshot plus events after their last sequence. Database notifications can accelerate updates but are not the durable queue.

### Process ownership and recovery

Adapters launch explicit executable/argument arrays with a chosen working directory and limited environment. Provider prompts travel through supported stdin/file mechanisms where practical. User-authored check commands declare their shell explicitly; do not build shell commands by concatenating agent output.

Track run, attempt, provider session, executable identity, process identity/start time, and workspace ownership. Use Windows Job Objects and Unix process groups behind OS-specific implementations to terminate the owned process tree. Test these behaviors on each platform; killing one parent PID is insufficient.

Persist launch intent before starting a process and reconcile uncertain launches. Never blindly replay an implementation after a crash: first inspect ownership and any existing session. UI reconnect and supervisor recovery are different guarantees. After a supervisor crash, mark uncertain attempts interrupted and require reconciliation; provider-specific resume may be available but is not universal.

If PostgreSQL becomes unavailable during a run, stop dispatching new work and approvals. Buffer bounded, redacted process events to a local recovery spool. At a safe boundary pause the run; if durable output cannot be retained, stop the owned process and mark recovery required. Replay spool events idempotently when the database returns.

## 7 Agent discovery and adapters

Provider identifiers are `codex`, `claude`, and `cursor`; show Claude Code as Anthropic's product. The Cursor editor launcher alone does not prove Cursor Agent is installed.

Discovery checks explicit configured paths first, then the OS executable search path and documented install locations. Inspect documented executable candidates such as `codex`, `claude`, and Cursor's `agent`/`cursor-agent`; verify identity through bounded version/help probes. Do not execute arbitrary matching files in the current repository or run recursive disk scans. Handle Windows executable extensions and command wrappers deliberately.

Store installation and runtime observations separately:

- Resolved executable and version, OS/architecture, and observation time.
- Authentication state: authenticated, unauthenticated, unknown, or probe failed.
- Capabilities: streaming, structured results, session resume, permission exchange, read-only execution, model discovery.
- Readiness: ready for a particular role, unavailable, incompatible, or needs verification.

A safe status probe should not install software, modify authentication, execute project work, or spend model credits. Authenticated status is only as current as the probe. Where no reliable non-generative probe exists, show unknown and let the user explicitly test connectivity.

Each adapter normalizes start, stream events, completion, cancellation, and supported resume into the internal contract. Provider-specific features remain explicit capabilities. Unknown stream events can be retained for diagnosis, but a missing/invalid terminal result must never count as success.

Codex documents non-interactive JSON event output; Claude Code documents programmatic execution and structured output; Cursor documents CLI automation. Exact flags and capabilities must be verified against the installed version and covered by adapter fixtures. [Codex](https://learn.chatgpt.com/docs/non-interactive-mode), [Claude Code](https://code.claude.com/docs/en/headless), [Cursor](https://cursor.com/docs/cli/overview).

Agents request product questions and submit findings through a scoped internal tool when supported, or a validated structured turn result when not. A structured-result question ends that turn and resumes with the saved answer/context. Do not assume every CLI accepts input while a turn is running. Malformed question/results get one bounded correction attempt, then a visible protocol failure.

### Refresh semantics

Provide `agents refresh` and a TUI refresh action. Rescan installations and supported safe status/capability probes, showing additions, removals, version changes, and timestamps. Allow explicit path edits when a running process inherited an old PATH.

Repository refresh re-reads local Git state. Fetching remote branches is a separate explicit operation; changing repository status does not silently change an active run's pinned base. Storage refresh tests the configured database connection without applying migrations. Memory reindex is a distinct operation from refresh.

Refresh never upgrades a CLI, restarts workers, changes model assignments, logs in, rewrites a plan, or changes an active run's configuration. New runs use updated discovery. Before starting a new turn, check whether its pinned provider is still compatible; block and request a recorded change if it is not.

Integration refresh updates connection/tool metadata for explicitly configured MCP servers. Source refresh retrieves a selected record into a new snapshot; neither silently alters the source material approved for an active task. New or changed tools remain disabled until their permissions are reviewed.

## 8 Repository identity and workspaces

Use Git queries to find the root, common Git directory, remotes, worktree identity, and current revision. Register a stable project ID with explicit checkout mappings. Remote URL normalization is a matching hint; it must not silently conflate forks, unrelated repositories, or projects with multiple remotes. Support repositories without remotes and nested repositories with an explicit project selection when ambiguous.

Resolve the starting revision in this order: explicit task selection, configured project remote/branch, selected remote's default branch. If unresolved, ask; never silently guess `main` or `master`. Record the resolved commit SHA and remote observation time. An explicit fetch policy can update remote-tracking refs before resolution; offline/stale mode must be visible.

Default to a detached Git worktree at that commit, avoiding an automatic commit or branch-management workflow. If the user chooses a task branch, validate it and reserve its worktree ownership. Clone mode creates an independent checkout in an approved managed directory. A clone is required only by explicit policy or a demonstrated incompatibility, not simply because another task exists. [Git worktree documentation](https://git-scm.com/docs/git-worktree).

Never reset, stash, clean, or switch the user's existing checkout automatically. Uncommitted work in that checkout is not implicitly part of a task based on a committed revision; including it requires an explicit snapshot choice recorded in the plan.

Assign one writer to each workspace. Record tracked and relevant untracked changes, deleted files, binary changes, base SHA, and content digests in artifact snapshots; no commit is required to review work. Exclude secrets and generated/dependency directories by policy. Handle submodules and LFS explicitly; stop for configuration when snapshot completeness is uncertain.

Worktrees do not isolate external services, ports, databases, or infrastructure credentials. Project setup can specify per-task resource names and approved setup/teardown commands. Such commands are executable configuration and require project trust. Workspace cleanup is explicit, restricted to recorded managed paths, and refuses to discard unaccepted changes.

## 9 Configuration and defaults

Use a versioned YAML schema with built-in defaults, user configuration, project configuration, and explicit run overrides. Show the effective value and origin. Repository settings cannot silently grant privileges withheld by user policy. Apply user restrictions after merging and require an explicit user decision to broaden permissions.

Persist a non-secret effective configuration snapshot on each run. Changes affect future runs; changing an active run requires a recorded action and, when material, renewed approval.

Proposed initial defaults:

| Setting | Default |
| --- | --- |
| Initial plan approval | Required |
| Final acceptance | Human |
| Step checkpoints | Progress visible; extra approval optional |
| Explicit disagreement | Ask human immediately |
| Missing required information | Ask human; no timeout acceptance |
| Correction cycles | Maximum three per step |
| Concurrent writing tasks | One |
| Workspace | Detached worktree from approved base |
| Git commits and remote writes | Manual |
| Database | Docker setup offered; connection string equally supported |
| Embeddings | User selects endpoint/model; keyword mode works without one |
| Cross-project retrieval | Off unless explicitly enabled |
| MCP memory access | Local and read-only |
| MCP context integrations | Optional, explicitly configured, read-only tools selected by the user |
| MCP credentials | Local OS credential store; explicit environment references supported |
| Integration configuration reuse | Non-secret settings in PostgreSQL; authenticate separately on each machine |

Models are configurable per role. Setup uses supported discovery where available, accepts an explicit model ID, and validates it where possible. Do not hardcode a universal model name or assume reasoning settings have identical meanings across providers. A provider-default model choice must be labeled as such, with the observed actual model recorded when available. Unsupported settings should be rejected, not ignored.

Configure wall-clock and output limits. Record token/cost usage only where the provider supplies trustworthy data; show unavailable otherwise. A cost limit cannot be advertised as strictly enforced when spend information arrives only after completion. Do not silently switch providers or paid authentication modes.

The initial setup uses the provider's existing CLI authentication without copying its login secrets. Store Crewweft-managed MCP credentials in the local OS credential store by default, with explicit environment references for managed/headless use. Database and embedding credentials can also use secret references. Keep all secrets out of repository configuration, run snapshots, provider subprocess environments unless specifically required, and searchable memory. If local credential storage is unavailable, ask for a supported alternative rather than silently persisting plaintext.

Store non-secret integration settings and logical credential references in PostgreSQL. Another PC can reuse those settings but must authenticate locally; shared configuration is not shared authentication. Do not persist MCP tokens in PostgreSQL or normal exports. Optional secrets-manager integration is deferred. See [credential storage and use on another computer](#credential-storage-and-use-on-another-computer) for token portability, OAuth, and disconnect behavior.

## 10 PostgreSQL and project memory

### Connection and provisioning

Use a normal PostgreSQL connection URL, for example a secret supplied through `DATABASE_URL`. Both local Docker and hosted databases use the same driver, schema, and migrations. pgx supports PostgreSQL URLs and connection parameters. [pgx connection configuration](https://github.com/jackc/pgx#connection-configuration).

Docker setup creates a dedicated pgvector-enabled PostgreSQL service with persistent storage and a configurable localhost port. Pin a supported image/version combination during implementation. A supplied connection string skips Docker entirely. Neon and Supabase both support pgvector; start with documented connection guidance rather than account-provisioning integrations. [Neon](https://neon.com/docs/extensions/pgvector), [Supabase](https://supabase.com/docs/guides/database/extensions/pgvector).

Require suitable TLS settings for remote connections and never silently disable certificate validation. Provide separate runtime and optional migration connection settings, since hosted poolers and restricted roles may not support schema changes or session-scoped behavior. Detect extension/schema privilege failures and show the required operator action. Do not require a PostgreSQL superuser for normal operation.

Apply explicit, versioned migrations under a migration lock; do not run them as a side effect of refresh. Test direct and supported pooled connections. Workflow correctness must not rely on a persistent session-level database lock. Multiple machines may share knowledge storage in future, but each first-release run belongs to a recorded local host and cannot be claimed by another host.

### Logical records

| Record group | Purpose |
| --- | --- |
| Projects and checkouts | Stable project identity and local repository mappings |
| Documents and revisions | Requirements, architecture, plans, and immutable approved versions |
| Runs and steps | Approved scope, effective config, status, dependencies, and continuation |
| Attempts and agent observations | Provider/session/process identity and capability observations |
| Events and messages | Ordered workflow history and role handoffs |
| Questions and answers | Missing information and human responses |
| Findings, responses, and decisions | Review arguments, disposition, approvals, and exceptions |
| Checks and artifacts | Commands, exit status, relevant output, snapshots, and content digests |
| Knowledge entries and source links | Decisions, summaries, architecture updates, and provenance |
| Integrations and source snapshots | Connection identity, tool permissions, source revisions, retrieval time, content hashes, and project mapping |
| Chunks, embeddings, and indexing jobs | Rebuildable retrieval data and retry status |

Original document text, summaries, decisions, and searchable evidence live in PostgreSQL. Large raw logs and patch/binary snapshots live in restricted local artifact storage with database references, checksums, sizes, and retention metadata. A remote database alone does not back up those local artifacts. Provide a documented database-plus-artifact backup and restore procedure.

### Knowledge lifecycle

Save records throughout the workflow. After implementation/review, draft a summary containing what changed, why, affected components, tests, remaining limitations, and proposed architecture updates. The final human decision includes those updates. Human-accepted knowledge remains distinct from changes actually integrated into the repository's default branch.

Tag knowledge as proposed, implemented in a workspace, reviewed, human-accepted, superseded, or rejected. Record project, task, document revision, base SHA, artifact digest, author role, timestamps, and source references. Do not overwrite old decisions when the architecture changes. Current-architecture views select accepted unsuperseded entries and expose applicability to the requested revision.

Search applies project/access/status filters first, combines PostgreSQL full-text results with pgvector similarity when available, and returns source references. Use exact fields for identifiers and paths as well as natural-language search. A result must show its status and whether it describes accepted work or a tentative attempt. [PostgreSQL full-text search](https://www.postgresql.org/docs/current/textsearch-intro.html), [pgvector](https://github.com/pgvector/pgvector).

Index asynchronously using durable jobs. Record embedding provider, model, dimensions, and chunking version. Do not compare vectors from different embedding spaces. Model changes create a new index generation; swap it into use only when ready. Indexing failure must not lose source records or silently block an otherwise complete task; show degraded semantic search and allow retry.

Source documents and repository files can contain instructions; retrieved content is context, never authority to change workflow policy or execute commands. Exclude configured secrets/files before storage/indexing and bound retrieved context. Users can export, correct, supersede, or delete knowledge, including derived embeddings and related retained artifacts under the retention policy.

## 11 MCP integrations and memory access

Crewweft has two separate MCP roles: a client gathering selected context from configured servers, and a server exposing scoped project memory. The official Go MCP SDK supports clients and servers. [Go MCP SDK](https://github.com/modelcontextprotocol/go-sdk).

The first-release intake is optional and read-only. A user can select a Jira issue or another source, discuss missing details in the terminal, and include a source snapshot in the plan. External content is evidence, not an instruction to launch work or change policy. The detailed design in [Integrations and local models](#14-integrations-and-local-models) defines source permissions, refresh, caching, and later internal deployment.

Expose the same memory service through an MCP stdio server in the first release. Proposed tools are `search_project_memory`, `get_project_architecture`, `get_decision`, and `get_task_history`. Bind access to explicitly allowed project IDs, limit result size, and return provenance and knowledge status. Do not expose arbitrary SQL, database credentials, workflow approval, or execution controls through the read-only memory server.

Local MCP-capable clients can launch the executable's MCP subcommand. Product questions remain a supervisor/TUI feature and do not depend on external clients supporting MCP elicitation.

Hosted ChatGPT/Claude integrations require a reachable authenticated transport and client-specific setup; a localhost stdio server alone does not provide that. Plan authenticated HTTP MCP as a later stage with access scopes and client compatibility checks. Do not assume that a hosted PostgreSQL database automatically makes the memory MCP service reachable.

## 12 Acceptance and verification

Every implementation stage has tests and a user-visible acceptance demonstration. Use unit tests for state transitions and policy, process fixtures for adapters, temporary real Git repositories for workspaces, and an actual PostgreSQL/pgvector service for persistence/retrieval integration tests.

CI must run native tests on Windows, macOS, and Linux. Cross-compilation alone is not sufficient evidence of process/terminal behavior. Live provider smoke tests are explicit because they need credentials and may use quota. Normal CI uses deterministic fake agent processes and redacted recorded stream fixtures.

Required release scenarios include stale plan approval, a disputed finding, an unanswered question, a provider update during a run, missing executable/authentication, a crash between launch intent and process registration, UI reconnect, supervisor restart, PostgreSQL outage, failed checks, dirty user checkout, modified review snapshot, interrupted indexing, and cross-project retrieval isolation. The implementation plan specifies when each is covered.

## 13 Decisions needed before implementation

1. Approve or amend this architecture and the linked implementation stages.
2. The name Crewweft is confirmed. Select the GitHub owner to establish the permanent repository and Go module path. Domain availability is outside scope.
3. Confirm the proposed first-release boundary: one writer at a time, optional read-only MCP context intake and local memory MCP included; hosted memory MCP and local generation runtimes follow later.
4. Confirm embedding setup: configurable endpoint/model with an explicit keyword-only starting mode. Database hosting and embedding hosting are independent choices.

The architecture does not authorize implementation by itself. Approval of a document revision authorizes only its agreed scope.

## 14 Integrations and local models

Crewweft gathers knowledge from its users and their tools. Later local model support will allow the complete workflow to run within an internal environment.

### Two MCP roles

Crewweft uses MCP in two directions:

```text
Users and project files ---------------------+
                                            |
Jira / other configured MCP servers -> Crewweft MCP client
                                            |
                               Selected source snapshots
                                            |
                          Plan -> Implementation -> Review
                                            |
                                Project memory in Postgres
                                            |
                             Crewweft memory MCP server
                                            |
                               External AI clients
```

The client gathers selected context. The server exposes Crewweft's own memory. They have separate configuration, credentials, tools, and access boundaries. The official Go SDK supports both roles. [Go MCP SDK](https://github.com/modelcontextprotocol/go-sdk).

### Gather information from people and systems

Planning accepts direct conversation, answers to questions, selected repository documents, and selected MCP records. For example, a user selects a Jira issue, Crewweft retrieves its description and relevant linked context, then asks which deployment environment the change targets. The issue snapshot, user answer, and resulting decision become traceable inputs to the plan.

Use integrations generically, with Jira as the first acceptance example. Avoid embedding a personal workspace, project key, or account ID into product defaults. A project configuration maps the local repository to its chosen external project and permitted sources.

Connection setup records a server ID, explicit stdio executable/arguments or HTTP endpoint, authentication method, secret references, project scope, allowed tools, timeouts, and result limits. Support the authentication and transport of the selected server and document it; an arbitrary MCP server is not automatically compatible. Refresh only configured servers. Do not scan the network or start MCP processes merely because a repository contains a configuration file.

Local stdio servers are executable software and require explicit trust before launch. Hosted-server authentication may require a browser-based login owned by that server even though Crewweft's product interface remains in the terminal. Select an internal/token-based integration when browser or external authentication is incompatible with deployment requirements.

For the first release, permit selected read operations only. Inspect tool schemas and maintain an explicit allowed-tool set; a server's read-only annotation is a hint, not proof. Unknown tools and changed schemas require review before use. Calls go through the supervisor's broker, which records the actual tool, parameters after redaction, source project, and outcome. Prefer tightly scoped integration credentials; do not give workers the integration's underlying secrets.

Creating Jira issues, adding comments, and changing status are future write operations. They require an explicit preview and human authorization for the actual action. Approving a coding plan does not imply permission to publish externally.

#### Credential storage and use on another computer

Persist integration configuration separately from authentication secrets. The configuration can be reused by another Crewweft installation through the same database or a configuration export; authentication readiness is specific to the current machine and user.

| Information | Storage and portability |
| --- | --- |
| Integration ID, server endpoint, auth method, project mapping, allowed tools | PostgreSQL and non-secret configuration exports |
| Logical credential reference | Configuration; resolves through the current machine's credential provider |
| API token and OAuth access/refresh credentials | Local OS credential store by default; excluded from database records and exports |
| Machine-specific executable paths and authentication status | Local machine settings/observations; revalidate on a new PC |
| Centrally managed credentials | Optional secrets-manager integration in a later stage |

During setup, explain where credentials will be stored and let the user choose the supported credential source. Offer the OS credential store by default and explicit environment-variable references for managed/headless environments. If a credential store is unavailable or locked, show that state and request another supported source; never silently save plaintext credentials in a file or PostgreSQL. Existing coding CLI logins remain owned by those CLIs rather than being copied into Crewweft.

A Jira API token can generally be reused on another computer while it remains valid and the account, scopes, and network policy permit access. For Atlassian's official MCP server, token authentication must be enabled by the organization and remains subject to IP allowlists. A different Jira MCP server may use a different credential format or scopes. [Atlassian API-token authentication](https://developer.atlassian.com/cloud/rovo-mcp/guides/configuring-authentication-via-api-token/).

For OAuth, default to a fresh authorization on each computer. Do not copy cached access/refresh tokens as part of configuration export or database reuse. Atlassian recommends OAuth for interactive use, and the resulting access depends on the user's consent and permissions. [Atlassian OAuth documentation](https://developer.atlassian.com/cloud/rovo-mcp/guides/configuring-oauth-2-1/).

The new-computer experience should be explicit:

```text
Jira integration found
Configuration loaded
Authentication needed: Sign in / Supply API token
```

After local authentication, revalidate the endpoint, identity, and permitted tools before marking the integration ready. A shared database must not make another machine appear authenticated. Missing local MCP executables are reported separately from missing credentials. Reusing a token does not transfer a running session or override source permissions.

Keep secrets out of RAG, model prompts, workflow events, connection URLs saved in configuration, command-line arguments, diagnostics, and configuration/knowledge exports. Inject a secret only into its intended connection or explicitly trusted MCP subprocess when that server requires it. Use masked entry for pasted secrets and avoid echoing them in errors. Disconnect removes the local credential; server-side token revocation is a separate, explicit action because it can affect other computers.

Do not implement credential storage in PostgreSQL for the first release, including encrypted token columns. Portable encrypted credentials would require a separate key-distribution and access-control design; storing the key alongside the ciphertext does not protect against access to both. A later secrets-manager adapter should let each machine authenticate to the vault and resolve an authorized secret reference. This is not automatic synchronization of OAuth refresh-token state; OAuth lifecycle handling needs its own design.

### Source snapshots and refresh

Store the selected useful content with server/source ID, external record ID, canonical URL, source version or update time when supplied, retrieval time, content hash, project, and importing identity. User answers carry the associated question and decision record. Agent interpretations remain proposals until accepted; importing a ticket does not establish that its description is correct or current architecture.

Keep capture bounded: requested records and relevant linked context, not a default bulk import of a company workspace. Apply project permissions, exclusions, redaction, and configured retention before indexing. Preserve source links in generated plans and search results. If sources disagree, show their provenance and ask the user rather than silently choosing one.

Integration refresh checks connection health and tool metadata. Source refresh retrieves a chosen record and creates a new snapshot when content changed. Active runs retain the source snapshots used for approval. A relevant requirements change creates a visible amendment; refreshing does not silently rewrite an approved plan.

Record partial results, pagination limits, deleted records, unavailable servers, and expired authentication. Cached content must show its age/status. On revoked access, disable retrieval of affected cached content until access is restored or the user confirms an independently authorized retained copy; retention and deletion policies apply to both stored source text and derived embeddings. First-release access is single-user; sharing the database later requires stronger per-user authorization.

External text is data. It cannot approve work, expand tool permissions, or instruct the supervisor to ignore its workflow. Tests must include a ticket containing instructions that attempt those actions.

### Local inference and an agent runtime

Keep three interfaces separate:

| Interface | Responsibility |
| --- | --- |
| Agent runtime | Complete a role's turn using tools, permissions, questions, and resumable state |
| Model client | Send inference requests to a model endpoint and consume its responses |
| Embedding client | Convert selected text into vectors for retrieval |

The initial Codex/Claude/Cursor adapters wrap complete agent runtimes. A later local model endpoint supplies inference, so Crewweft also needs a tool loop: build context, call the model, validate requested tools, execute allowed actions, return results, and stop or ask a question within configured limits. An alternative is a compatible local agent runtime that already performs this loop, provided its controls can be tested.

Ollama is one candidate endpoint to evaluate, not a selected dependency. Its documented API supports local inference, but it also supports cloud models, including through a local server; therefore a localhost URL alone does not prove local execution. [Ollama API documentation](https://docs.ollama.com/api/introduction).

Assign local or hosted runtimes independently to planner, implementer, and reviewer. Keep the same approved-plan gates, evidence requirements, disagreement escalation, and memory semantics. Capability checks should cover structured responses/tool calls, usable context size, cancellation, and role suitability. If a model cannot perform a required role, report that limitation instead of relaxing verification or silently routing to a cloud provider.

Model selection needs hardware and task measurements. Record endpoint/runtime, model revision, relevant context configuration, and observed results. Use smaller retrieved context when appropriate, but preserve the approved requirements and unresolved findings. Local embedding generation is independent of local coding inference and may arrive earlier.

### Internal operation

Define deployment intent explicitly:

| Mode | Permitted services |
| --- | --- |
| Connected | User-approved local and hosted services |
| Internal | Only approved machine/private-network services, enforced by deployment controls |
| Offline | Same-machine services and staged assets; no external network required during execution |

An internal deployment needs internal inference, embeddings, PostgreSQL, MCP connectors, artifact storage, and any Git/package/test dependencies used by agents. A locally launched connector can still call a cloud service; its downstream destinations matter. Cloud-hosted Jira is a connected deployment source even when the MCP process runs locally. An internal Jira or another internal source can be configured when available.

Internal mode must never fall back to cloud models, hosted embeddings, external search, or external storage on failure. Make update checks, telemetry, error reporting, and model downloads explicit and disabled in that mode. Prestage installation assets for offline operation. Hosting Crewweft's source on GitHub does not require an installed instance to contact GitHub during a task.

Application endpoint validation covers Crewweft's own calls; it cannot alone contain arbitrary provider or shell subprocess networking. Verified internal operation therefore requires an OS/container/network boundary covering all owned processes and services, with allowed destinations tested. Endpoint trust must include redirects and downstream service behavior. Do not label a run internal merely because its configuration says so.

### Delivery order

1. Preserve runtime/model/embedding and integration interfaces in the initial design.
2. Add optional read-only MCP intake and source snapshots before connecting real coding agents to approved plans.
3. Include provenance and source status in memory search, then expose local read-only memory MCP.
4. Add local model runtimes in a separately approved follow-up, using the same workflow contract.
5. Validate an entire run with external egress blocked, including tools, embeddings, integrations, and recovery. Only then claim support for fully internal operation.

The first release remains a terminal application using installed coding CLIs. These additions prepare a clear path to internal operation without implying that local model execution already exists.
