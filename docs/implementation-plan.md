# Crewweft implementation plan

Status: draft awaiting approval. Date: 2026-11-21. Prerequisite: approve [architecture](architecture.md) and choose the initial release boundaries. No implementation stages have started.

## Delivery rules

Follow stages in order. Each stage ends with a working demonstration, relevant passing tests, a self-review, and an updated progress entry. Keep the user informed even when intermediate approvals are disabled. Stop for a plan amendment if a stage changes architecture or scope.

Implementation and review are separate roles. Normal corrections return to implementation; an explicit disagreement goes to Lasha. Do not commit, push, merge, or create PRs. These documents define the proposed scope; implementation requires explicit approval.

Use standard Go tests from the first code change. The empty workspace currently has no test setup; stage 1 creates it as part of the proposed implementation. Before starting each later stage, identify its regression/behavior tests, then implement and run them. Review for unused code, swallowed errors, unclear ownership, goroutine/process leaks, inconsistent naming, sensitive logs, and unnecessary scope.

## Proposed source layout

```text
cmd/crewweft/            executable and subcommand wiring
internal/app/           application operations and typed requests
internal/config/        schema, precedence, validation, secret references
internal/credentials/   OS credential store and environment-reference resolution
internal/domain/        workflow states, approval rules, typed events
internal/supervisor/    dispatch, ownership, recovery, event delivery
internal/transport/     local authenticated API and event stream
internal/agents/        discovery, capability registry, shared adapter contract
internal/agents/codex/  Codex adapter
internal/agents/claude/ Claude Code adapter
internal/agents/cursor/ Cursor Agent adapter
internal/integrations/ MCP client connections, tool policy, source snapshots
internal/process/      platform-specific process creation/cancellation
internal/workspace/    Git detection, worktrees, clones, snapshots
internal/checks/       configured verification command runner
internal/storage/      PostgreSQL queries, transactions, migrations
internal/memory/       ingestion, provenance, search, indexing jobs
internal/tui/          setup, conversation, plan, activity, inbox, memory
internal/mcp/          scoped read-only memory server tools
internal/testutil/     fake agent executables and reusable fixtures
migrations/            explicit versioned SQL migrations
testdata/              redacted provider streams and test documents
deploy/                local database Compose configuration
docs/                  architecture, plan, operations, compatibility, progress
```

Create packages as their stage needs them; avoid empty scaffolding for every directory. Use `crewweft` as the executable name; keep the permanent module path pending until the GitHub owner is selected.

## Stage 0 Approve the design

**Deliverables:** approved architecture/plan revision, explicit release boundary, embedding setup decision, and GitHub owner/module path decision. The name Crewweft is confirmed. Domain checks are not required.

**Acceptance:** Lasha explicitly approves implementation scope. Do not rename the workspace or reserve external names as an implicit part of documentation approval.

## Stage 1 Establish the Go terminal application and configuration

**Areas:** `cmd/`, `internal/config/`, `internal/domain/`, `internal/tui/`, Go module files, initial test setup.

**Work:** choose compatible Go and library versions; build the executable, help/version commands, basic TUI navigation, versioned configuration, default/user/project/run precedence, secret references, and effective-configuration display. Add placeholder views that clearly show unimplemented behavior rather than simulated success. Implement core approval/transition rules as pure domain logic.

**Tests before implementation:** table-driven tests for allowed/forbidden transitions; missing/stale approval; config precedence, invalid fields and role settings; user restrictions surviving project overrides; secret redaction; keyboard navigation and resize state handling.

**Acceptance:** a native binary opens on all three operating systems; help/config validation work without PostgreSQL; an unapproved task cannot enter implementation even through direct application calls.

## Stage 2 Build agent discovery and refresh

**Areas:** `internal/agents/`, `internal/process/`, Agents view, doctor command, test utilities.

**Work:** discover explicit paths and trusted PATH candidates for Codex, Claude Code, and Cursor Agent. Add bounded version/help/status probes, normalized capability observations, timeout/error handling, and refresh diff output. Separate the Cursor editor launcher from the actual agent. Capture a compatibility matrix based on tested versions and native OS behavior.

**Tests before implementation:** fake executable absent/present/duplicated, spaces and Unicode in paths, Windows executable wrappers, executable identity mismatch, hanging probe, malformed version, authentication unknown, added/removed/upgraded executable, cancelled refresh, and no agent generation during discovery.

**Acceptance:** install/remove a fake provider between refreshes and see the list change without restarting the UI. Existing observations retain timestamps and failures are visible. No paid provider turn is needed to pass the stage.

## Stage 3 Add PostgreSQL provisioning and durable records

**Areas:** `internal/storage/`, `migrations/`, `deploy/`, terminal setup.

**Work:** offer local Docker PostgreSQL with pgvector or an existing connection string. Add connection checks, TLS options, separate migration/runtime settings, versioned migrations and locks, core workflow tables, append-only events, and atomic state/event writes. Define local artifact paths and backup boundaries.

**Tests before implementation:** real PostgreSQL integration tests for clean installation, repeated migration, concurrent migration attempts, transaction rollback, event ordering/deduplication, restricted runtime role, missing pgvector/privileges, incorrect credentials, unavailable server, and credentials redacted from errors. Test connection-string parsing and supported pooler configuration without relying on session locks.

**Acceptance:** the same application stores/reloads a draft with Docker or a provided connection string; supplied connections do not start Docker. Setup/doctor explains failures without launching work or destroying existing data.

## Stage 4 Implement the local supervisor and reconnectable terminal

**Areas:** `internal/supervisor/`, `internal/transport/`, `internal/process/`, TUI connection handling.

**Work:** per-user supervisor lock, background startup, authenticated loopback transport, durable event cursors, run ownership, bounded logs, pause/cancel operations, and a recovery spool for database outages. Use fake long-running child agents to establish process guarantees before real provider integration.

**Tests before implementation:** duplicate launch requests, competing supervisor startups, UI disconnect/reconnect, wrong/missing local token, disallowed origins, bounded output, redaction, process tree cancellation, PID reuse protection, crash around launch registration, database disconnect/replay deduplication, and spool exhaustion.

**Acceptance:** closing/reopening the TUI reconnects to a still-running fake task. Cancelling terminates owned child processes on Windows/macOS/Linux. Supervisor restart marks uncertain work interrupted and does not launch a duplicate writer.

## Stage 5 Add repository detection and isolated workspaces

**Areas:** `internal/workspace/`, project records, Projects view, artifact storage.

**Work:** identify repositories/worktrees, register checkouts, select remote/base revision deterministically, create managed detached worktrees and explicit independent clones, preserve dirty user checkouts, and capture reviewable snapshots without commits. Record setup/check commands and resource configuration.

**Tests before implementation:** temporary Git repositories covering subdirectory discovery, nested repos, existing worktrees, no remote, multiple remotes, missing default branch, stale refs/offline mode, dirty tracked/untracked files, branch/path collisions, spaces/Unicode paths, symlink/path containment, deleted/binary files, interrupted clone, and cleanup refusal for unaccepted changes.

**Acceptance:** task preparation records the exact base SHA and creates a separate workspace while leaving the original checkout unchanged. Ambiguous base selection asks a question. Snapshot content includes all supported source changes or reports unsupported cases.

## Stage 6 Implement planning documents and the human inbox

**Areas:** `internal/app/`, `internal/domain/`, document/question storage, Plan/Inbox views.

**Work:** versioned document/plan authoring and import/export, acceptance criteria, ordered steps, approval bound to revisions/config/base, structured questions, free-text answers, and recorded continuations. Enable discussion/planning without granting implementation permission. Refresh and concurrent UI actions must not bypass stale-revision checks.

**Tests before implementation:** missing required plan fields, approval of a stale version, document edit after approval, double approval/answer submission, unanswered questions, cancelled questions, answer requiring replanning, role spoofing, and attempts by a fake agent to self-approve.

**Acceptance:** the user can inspect/approve a plan entirely in the terminal. A blocking question pauses its dependent work and survives restart. A changed approved specification requires approval again.

## Stage 6A Add optional MCP context gathering

This stage is inserted before provider execution because gathered sources become inputs to the approved plan. It does not require an external integration for ordinary local-project use.

**Areas:** `internal/integrations/`, source-snapshot storage/migrations, Integrations view, planning context selection.

**Work:** configure explicit MCP servers over local stdio or supported authenticated HTTP; discover tool schemas; allow selected read tools per project; gather a selected Jira issue or equivalent source; preserve source identity, retrieved version/time, and content hash. Combine this with direct user answers and selected repository documents. Keep source refresh separate from tool-metadata refresh. Show changed source content and require plan amendment when it affects approved requirements. Route tool access through the supervisor rather than sharing integration credentials with every agent.

Add a credential-provider interface with OS credential storage by default and explicit environment references. Persist/export only non-secret connection settings and logical secret references. Show the storage choice during setup, support masked token entry or supported OAuth authorization, and require authentication again on a new machine. Keep local disconnect separate from provider-side revocation. Never fall back to plaintext files/database columns when the credential store fails.

**Tests before implementation:** fake MCP server for tool discovery changes, unknown/write tools rejected, expired auth, timeouts, pagination, partial results, source links and content hashes, duplicate retrieval, changed source after approval, sensitive fields excluded, source text attempting to issue instructions, project access isolation, and disconnected-source cached/stale labeling. Prove external writes cannot run through the read-only broker.

Credential tests cover missing/locked/unavailable stores, environment resolution, OAuth expiry/reauthorization, machine-specific readiness, disconnect without global revocation, and reusing non-secret configuration in a second isolated machine profile. Use synthetic canary tokens to verify they do not appear in database rows, logs, prompts, process arguments, exports, or indexed content. Verify native OS store operations with disposable test entries and cleanup; use a fake store in routine unit tests.

**Acceptance:** select a ticket, gather its details, answer one missing-information question, and approve a plan citing that ticket snapshot. Re-fetching a changed ticket displays the difference without silently changing the approved plan. Verify one selected Jira MCP server when credentials are available; record its actual server/version/auth support instead of claiming every Jira server is compatible.

Load the same integration settings in a second machine profile: it must show authentication needed until a local credential is supplied or OAuth authorization completes. Configuration export must contain no tokens. Native credential-store verification is required on each supported OS.

## Stage 7 Integrate the three coding agent adapters

**Areas:** provider adapters, process manager, structured question/result parser, Activity view.

**Work:** implement one adapter at a time against the shared contract, beginning with a locally available provider. Support bounded streaming, session identity, explicit model settings, supported permissions, validated results, cancellation, and capability-gated resume. Convert provider/internal question requests to persisted inbox items. Use explicit handoff context if resume is unavailable; identify it as a new session.

**Tests before implementation:** redacted recorded streams plus fake executables for success, nonzero exit, success exit with invalid result, truncated JSON, unknown events, missing final response, provider auth failure, rate limits, timeouts, unavailable model, required unsupported capability, question/answer continuation, and provider version changes between turns.

**Acceptance:** each supported adapter completes a small approved task in a disposable repository, emits visible progress, and can be cancelled. Record native platform/version evidence; unavailable credentials are a named test gap, not a passing live test. Refresh never replaces the adapter configuration of a running attempt.

## Stage 8 Add independent verification and the review loop

**Areas:** `internal/checks/`, findings/response storage, review routing, artifact snapshots, Inbox/Activity views.

**Work:** run approved checks independently of agent claims; bind results to snapshots; launch a fresh reviewer with the plan and evidence; route accepted findings to implementation; immediately escalate explicit disputes; enforce bounded correction cycles; implement per-step and final whole-task review.

**Tests before implementation:** agent falsely claims tests passed, actual check failure/timeout, test-induced source mutation, stale review digest, missing/untracked file in snapshot, normal correction and rereview, explicit dispute creating a human item, human dismissal with rationale, no-progress repair loops, and final review finding an interaction across steps.

**Acceptance:** demonstrate a fix cycle and a disagreement cycle. The first reaches review success after real checks; the second waits for the human. Acceptance cannot use a review from an older snapshot. Human exceptions remain visible in the outcome.

## Stage 9 Add project memory and semantic retrieval

**Areas:** `internal/memory/`, knowledge/embedding migrations, Memory view, background indexing jobs.

**Work:** ingest approved documents, user answers, selected MCP source snapshots, decisions, task summaries, review outcomes, and selected evidence. Preserve separate statuses for source facts, user decisions, and agent proposals. Implement status/provenance filters and PostgreSQL full-text search first. Add configurable embedding endpoint/model support, pgvector retrieval, result fusion, cited sources, retryable indexing, index generations, and explicit reindex. Add architecture-update proposals to final acceptance.

**Tests before implementation:** deterministic fake embeddings with real pgvector; project isolation; exclusion/redaction; identifier keyword retrieval; known semantic retrieval examples; source citation correctness; tentative versus accepted status; superseded decisions; embedding dimension/model mismatch; interrupted/retried jobs; reindex generation switch; deletion removing derived chunks; and missing local artifact reporting.

**Acceptance:** a later session retrieves a previous accepted decision and implementation with source references. Rejected work is not presented as current architecture. Embedding failure leaves original records and keyword search usable. Show one configured real embedding endpoint smoke test separately from deterministic CI.

## Stage 10 Expose local read-only memory MCP

**Areas:** `internal/mcp/`, scoped memory application service, client configuration documentation.

**Work:** implement stdio MCP tools for project memory search, architecture, decisions, and task history through the same service as the TUI. Require explicit allowed projects; enforce result limits and source/status output. Leave hosted transport and writes for a later approved stage.

**Tests before implementation:** protocol-level requests with the official Go SDK test client, invalid inputs, disallowed projects, oversized result requests, missing records, disconnected database, clean stdout protocol framing, and absence of execution/approval/SQL tools.

**Acceptance:** an MCP client retrieves the same scoped decision as the terminal. Document tested clients and avoid claiming universal Claude/ChatGPT compatibility. No database credential is exposed to the client.

## Stage 11 Complete setup, operational documentation, and release verification

**Areas:** TUI integration, packaging configuration, compatibility/operations documentation, end-to-end tests.

**Work:** join setup/discovery/planning/execution/review/memory into one coherent terminal flow. Add configuration export, explicit database migration/recovery commands, backup/restore instructions for database and local artifacts, and release build recipes. Verify Windows/macOS/Linux native behavior and document supported architectures/provider versions.

**Tests before implementation:** full fake-provider scenario covering optional MCP ticket intake, plan approval, a question, implementation, failed check, correction, review disagreement, human decision, final acceptance, restart, and memory retrieval citing the original ticket snapshot. Test both a changed source and an unavailable integration without silently altering approved scope. Repeat with a supplied PostgreSQL URL. Test backup/restore, fresh install, schema upgrade from the previous test fixture, and output with no color/narrow terminal.

**Acceptance:** all required checks pass on the supported native platforms; a user can complete the workflow without a browser UI. Demonstrate live provider combinations available to the release tester and list any unverified combinations. Do not call the release complete while required native/provider verification remains unresolved.

## Planned follow-up A Add local model runtimes

**Scope:** after the first release and separate scope approval. See [Integrations and local models](architecture.md#14-integrations-and-local-models).

**Work:** add configurable local/internal inference endpoints behind a model-client interface, then a tool-using runtime implementing the same role contract as CLI agents. Reuse the supervisor's file/check/question/MCP tools and policy. Support model discovery/refresh on configured endpoints, context limits, structured results, cancellation, and independent local embedding configuration. Select endpoint software and model versions using measured role capability and available hardware.

**Tests and acceptance:** fake inference server tests for malformed tool calls, repeated calls, bounded loops, unavailable model, context overflow, cancellation, and no cloud fallback. Run the same approved-plan implementation/review/disagreement scenario with real local models; record runtime, model revision, hardware, and failures. A model endpoint responding to chat is not sufficient evidence of coding-agent support.

## Planned follow-up B Verify fully internal operation

**Scope:** depends on local model runtimes; no claim of fully internal operation before this passes.

**Work:** define an explicit internal endpoint policy for inference, embeddings, PostgreSQL, MCP, Git/dependency sources, and artifacts. Disable cloud fallback and optional outbound services. Provide an operational network boundary for spawned tools as well as Crewweft's HTTP calls. Prestage required models/images/dependencies for disconnected use; keep GitHub publishing separate from runtime requirements.

**Tests and acceptance:** execute setup from staged assets and the full planning/question/implementation/review/memory workflow with external egress denied. Verify attempts to use external models, redirects, connectors, package sources, and hidden provider fallbacks fail visibly. Distinguish same-machine offline testing from private-network testing with internal services. Record the supported OS/runtime isolation method and actual network evidence.

## Later stages requiring separate scope approval

- Multiple concurrent writers with dependency scheduling and explicit integration ownership.
- Authenticated remote MCP for hosted AI clients and optional proposed memory writes.
- Optional secrets-manager credentials for multiple computers, with per-machine vault authentication and separately designed OAuth lifecycle handling.
- Multiple users, permissions, and coordinated execution across machines.
- Full terminal attachment via PTY/ConPTY where provider interaction requires it.
- Hosted-database provisioning helpers and embedding model installation.
- Optional additional coding agent providers and provider-routing policies.

## Progress record template

For each completed stage record: approved scope/revision, changed areas, demonstrated behavior, tests and actual outcomes, review findings and resolution, known limitations, and the next stage. A proposed test is not a passed test; a build is not proof of native execution.

## Current progress

Only the architecture, implementation plan, and README exist. No Go module, implementation, database, agent session, or test suite has been created. Implementation awaits approval of stage 0.
