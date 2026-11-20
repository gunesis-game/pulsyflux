
# Project steering: pulsyflux

Read this file first. It owns the mission, decisions and working rules for the repo. When the repo is created, this file moves to the repo root as `steering.md` in the first scaffold commit.

This file is self-contained. It does not depend on any other repo or steering file.

### Working rules

- All work is performed directly on `main`. Never create or use another branch.
- Do not create GitHub Issues or pull requests.
- The current phase is steering/design only. Do not implement software or scaffold the repository until the user explicitly authorises implementation.
- Update this steering file as decisions are agreed; do not silently treat proposals as settled decisions. Every rule needed to build the project is written here.
- Think at both subsystem and whole-system levels. Whenever a mechanism or policy is designed for one subsystem, assess where else it applies and whether a consistent cross-cutting approach is appropriate. Reuse the underlying principles where suitable, but explicitly account for each subsystem's semantics, failure modes and configuration rather than blindly copying local defaults. In particular, review retry and reconnect behaviour across the architecture for backoff, jitter, caps, reset conditions and failure handling.

## Mission

Build one general-purpose, stream-based messaging infrastructure in Go. A single Broker owns streams, their message queues, publishing, subscriptions, fan-out delivery, retention, replay and message lifecycle. Application, Security, Tracing and Admin are independent messaging participants: each publishes to and subscribes to the streams required by its responsibilities. Components communicate with one another through Broker streams; there is no direct component-to-component messaging or RPC layer. Application is the client-facing participant; Security and Admin are internally provisioned; Tracing participates internally and also supports its documented external access/integration surface. Core supplies shared foundational and snapshot/persistence capabilities applied to Broker-owned stream state. Each stream is identified by the hash of a JSON Schema. The whole public surface is generated from a single AsyncAPI document. Applications that need messaging use this bus instead of building their own.

The bus runs only on the user's own machine. It is never exposed to the internet and never uses a cloud service.

## Current status

The project remains in architecture/design only; no software implementation or repository scaffolding has been authorised. The GitHub repository exists, and this `steering.md` is the authoritative, self-contained source for the project's mission, requirements, decisions, working rules and acceptance criteria.

Keep those requirements and criteria in this file; do not create a separate `requirements.md`.

Any unresolved decision that changes scope, externally visible behaviour or an acceptance criterion must be settled here before implementation. Do not introduce decisions in implementation that are absent from this file.

## Authority order

1. `api/asyncapi.yaml`, the only public contract.
2. This `steering.md`.
3. The owning spec under `docs/` or `specs/`.
4. Generated code in `generated/`.
5. Hand-written code and tests.

If two of these disagree, the higher one wins and the lower one is fixed.

## Decisions

## Foundational architecture: Core, Broker, streams and participants

This is the governing architecture for every subsystem. Subsystem-specific descriptions, package boundaries, examples and API decisions must conform to it. Do not create a second communication or storage architecture for an individual subsystem.

### Core

Core provides foundational capabilities shared across the system, including lifecycle and shutdown coordination, shared execution context, configuration, resource ownership, and reusable snapshot/persistence policy and mechanisms. Core is not a second message bus and does not own a parallel copy of Broker messages.

Core's snapshot and persistence facilities operate on state owned by the relevant subsystem. For stream message state, Broker is the authoritative owner of the stream, queue and published messages; Core supplies the shared snapshot/persistence capability and policy applied to that Broker-owned state. There must not be duplicate queue histories, per-subscriber message stores, or subsystem-specific persistence stores for data already owned by Broker.

### Broker

Broker is the one shared messaging and stream infrastructure. It owns registered streams and their authoritative message queues/logs, accepts publications, manages subscriptions, performs fan-out routing and dispatch, enforces delivery/backpressure rules, and owns message ordering, retention, replay and stream lifecycle. Broker also owns the message data from which stream snapshots and durable persistence are produced. Snapshotting and persistence must not introduce a second authoritative message store.

A stream is the communication boundary. Components publish messages to streams and subscribe to streams. A publisher does not need to know which components subscribe; a subscriber does not need to know which component published. One stream may have many publishers and many subscribers. Fan-out semantics mean each authorised subscriber receives its own delivery of each message; this is not competing-consumer work-queue semantics.

Internal and external messaging use the same Broker stream model and core messaging semantics. External access is an exposure and authorisation boundary, not a separate messaging architecture. Transport adapters may differ, but they must not create parallel stream, queue, subscription or persistence models.

### Messaging participants

Application, Security, Tracing and Admin are independent participants in the same infrastructure. Each may publish and subscribe, subject to its role, identity, scope and authorised stream relationships. Their responsibilities differ; their messaging infrastructure does not.

- **Application:** the client-facing/external application participant. External clients use the documented Application-facing API and its approval and stream-scope rules. Application may publish to and subscribe to streams required by its application responsibilities.
- **Security:** an internally provisioned participant. It publishes security and authorisation outcomes and subscribes to streams carrying requests and events it must evaluate. Its policy decisions remain Security-owned; Broker transports the messages but does not make those decisions.
- **Tracing:** an internal participant that subscribes to relevant source streams and publishes tracing/observability events to appropriate stream(s). Its external trace access or integration surface consumes the same Broker-owned stream data under the documented authentication and authorisation rules. Tracing does not own a parallel message history, queue or persistence engine.
- **Admin:** an internally authenticated administrative participant. It publishes administrative instructions/events and subscribes to streams needed for administration and observation. Its global administrative authority comes from its trusted identity and role, not from a separate messaging path.

These are role and responsibility distinctions, not separate broker implementations. A participant may both publish and subscribe. Exact stream permissions remain governed by authenticated identity, role and scope.

### Communication rule: no direct component-to-component calls

Application, Security, Tracing, Admin and other domain components must not communicate by directly calling one another's methods, RPC endpoints, private channels or bespoke integration interfaces when exchanging domain messages or requesting domain action. They communicate by publishing to and subscribing from Broker streams. The receiving component interprets and processes the message according to its own responsibility and may publish a resulting message to another stream.

A publisher must not depend on knowing its subscribers. A subscriber must not depend on knowing the publisher's implementation. Broker owns delivery; domain components own message meaning and processing. Ordinary constructor dependency injection and calls to shared Core infrastructure are not domain messaging and remain valid.

Do not add a generic component-to-component messaging abstraction. Request/decision workflows, including Security authorisation, are logical messages carried on streams and use ordinary Broker publishing and subscription. HTTP/SSE is the documented external transport/API surface; it does not create a private direct-call channel between internal components.

### Streams, queues, snapshots and persistence

A stream is a schema-identified, ordered message history managed by Broker. Its queue/log is the authoritative in-memory collection of published messages and the source for fan-out, replay, retention and gap reporting. Per-subscriber delivery queues are bounded delivery buffers; they are not independent stream histories and do not own the published messages.

Core's shared snapshot/persistence facilities operate on Broker-owned stream and message state. A snapshot captures a defined Broker stream/queue state or checkpoint; persistence retains configured Broker-owned message data according to the stream-class policy and any explicit per-stream override. Restoring persisted data repopulates Broker's authoritative stream state before that state is served. Do not create a separate stream store in Tracing, Security, Admin or Application.

The exact durability guarantees, snapshot boundary, persistence ordering, crash-loss window, restoration validation and reconciliation must follow decisions recorded in the relevant sections below. This architecture statement does not silently change already-settled retention, authentication-state, Security recovery or trace-event contract decisions.

### Architectural review rule

Whenever a subsystem proposal introduces a queue, event bus, stream, history, snapshot, persistence worker, replay mechanism, subscription registry or component integration, first determine whether Broker or Core already owns that capability. Reuse the shared capability where semantics match. If a subsystem genuinely needs additional domain state, distinguish that state from Broker-owned messages and justify why it cannot be represented in its own stream. Never duplicate infrastructure merely to simplify one subsystem locally.


### Deployment

Local machine only. See Local-only rules.

### Contract format

AsyncAPI 3.1.0 (latest release of asyncapi/spec).

One file: `api/asyncapi.yaml`.

### Schema dialect for streams

JSON Schema 2020-12 (latest published version on json-schema.org).

### Go version

Pin the selected Go toolchain in `go.mod` and in CI.

The `go` directive records the minimum Go version required by the module. Use the `toolchain` directive or the CI setup to pin the exact toolchain.

Do not describe a version as both "latest stable" and permanently fixed.

### Code generator

Our own generator in `cmd/contractgen`, run through `go generate`.

See Why a custom generator.

### Transport

- HTTP for every request/response operation.
- Server-Sent Events (SSE) for every `receive` operation.

### JSON Schema library

`github.com/santhosh-tekuri/jsonschema/v6`.

Pure Go and supports 2020-12.

### Canonical JSON

RFC 8785 (JCS) through `github.com/gowebpki/jcs`, pinned.

### Storage

Broker owns every stream's authoritative message queue/log and the published messages it contains. The queue/log is the source of truth for stream ordering, fan-out, retained history, replay and message-gap calculation. Per-subscriber delivery queues are bounded delivery buffers only; they do not own or duplicate stream history.

Core provides the shared in-memory snapshot and persistence model used for Broker-owned stream state. Every stream has a complete in-memory snapshot governed by its JSON Schema, regardless of whether any part is persisted. The snapshot represents the corresponding Broker-owned stream/queue state; it is not a second message store. The persistence policy selects which schema-defined properties are written to durable storage. A stream's role determines its stream class and class-level persistence policy; individual streams may override that policy. A class with no explicit persistence policy is non-persistent by default. Snapshot availability in memory is never conditional on persistence.

Broker remains the owner of stream identity, queues, messages, ordering, retention, subscriptions and replay. Core owns the reusable snapshot/persistence mechanism and policy. Snapshot and persistence work from Broker-owned messages and stream state; restoring persisted data restores that state to Broker before it is made available for normal publication, subscription or replay. No subsystem may create an independent queue, message history, file store or duplicate state store for data already represented by Broker streams and Core snapshots.

Application, System, Security and Admin stream state follows the shared Core snapshot/persistence policy. This does not make all control-plane or authentication state durable. Security state, including subscriber identities, roles, credential hashes, pending approvals and authorisation state, is represented in the shared in-memory snapshot model according to its schema. Security snapshots are not persisted; sensitive and transient Security properties are excluded from durable storage by policy. Plaintext bearer credentials are not retained in snapshots; credential hashes are held where required for verification. This does not change the existing authentication/authorisation lifecycle or bootstrap rules.

In-memory snapshots are lost when the process exits. Only schema properties selected by an explicit persistence policy may be restored after restart. Authentication and authorisation state remains non-persistent. Persistence and snapshots must not create a second authoritative message history beside Broker's queue/log.

### Docs UI

Served at `/`, with a "Try it" console generated from the contract.

### Tracing

### Governing architecture — Tracing is a Broker participant

Tracing is an internally provisioned System-role messaging participant. It subscribes to relevant source streams and publishes trace/observability events to the configured tracing stream or streams. It may also expose trace data through the documented external trace API and monitor UI. Internal observation and external trace access use the same Broker-owned stream data and the existing authentication/authorisation model; neither creates a separate trace store or messaging path.

Tracing is both a publisher and a subscriber. It uses the ordinary Broker publishing and subscription mechanisms, ordinary identity and credential verification, and the standard Security authorisation workflow. Its System identity does not grant a bypass, unlimited queue, special delivery semantics or direct access to another component's implementation. Application, Security, Admin and future participants interact with Tracing only through streams and documented external interfaces.

Broker owns tracing-stream registration, message queues/logs, message ordering, fan-out, bounded delivery queues, retention, replay, gap reporting and message lifecycle. Core's shared snapshot/persistence mechanism applies to tracing streams according to the stream class and explicit persistence policy. Tracing does not own a parallel per-stream history, queue, recorder store, snapshot model, persistence worker, trace file rotation system or restore path. The trace API reads Broker-owned tracing-stream messages; it does not tail files or read a separate trace database.

### Trace events

Trace events are ordinary messages published to a tracing stream and validated by that stream's JSON Schema. The TraceEvent schema remains part of api/asyncapi.yaml and its Go type is generated like every other wire type. Trace events use the ordinary Broker message envelope and are subject to the same publication validation, stream ordering, retention, replay, persistence policy and subscriber delivery rules as other messages.

Trace records describe what happened to publishes and subscriptions, in the order observed. They may include event kind, timestamp, source stream ID, subscription ID, message sequence/correlation metadata, and kind-specific diagnostic details. Message payload data and bearer credentials or credential hashes are never copied into trace records.

The existing domain event kinds remain:
- schema.registered: a new schema is admitted, not on an idempotent re-registration.
- publish.accepted: a message is appended; details may include its size in bytes.
- publish.rejected: a publish to a known stream fails payload validation or the configured message-size limit; an unknown-stream request is represented by its HTTP error because no stream exists to carry a trace event.
- subscription.opened: a subscriber connects; details may include its supplied resume cursor.
- delivery: a message frame has been written and flushed to a subscriber.
- subscription.gap: a gap/control condition is reported to a subscriber.
- subscription.overflow: a subscriber's bounded delivery queue overflows and it is detached.
- subscription.closed: a subscription ends; details identify the reason where safe.
- Authentication and authorisation outcomes may be recorded as metadata, including valid decisions and late, duplicate or unknown decision diagnostics. They must not expose credentials, credential hashes, message payloads or administrator-only information to unauthorised readers.

Tracing is observational. It is not part of the Security authorisation decision path, does not make policy decisions, and must not block publishing or delivery. Tracing must not recursively instrument its own trace publication without an explicit recursion-prevention rule. Tracing's own source subscriptions use the ordinary bounded queue, detachment and gap behaviour; no unlimited queue or special broker privilege is permitted.

### Ordering, cursors and gaps

Broker's per-stream message sequence and run-scoped message identity are authoritative for ordering, retention and replay. A tracing stream is an ordinary Broker stream. There must not be a second trace-only message history or replay engine.

The earlier trace-specific TraceEvent.traceSeq, traceSeq-based Last-Event-ID cursor, trace.gap event and independent trace retention rules require reconciliation with Broker's authoritative stream seq, standard Last-Event-ID cursor, stream.gap event and retention rules. Preserve any trace-specific information that is genuinely required by the public trace contract, but it must map consistently to Broker message identity and must not create a second ordering, retention or replay source of truth. This mapping is an explicit pre-implementation contract decision; do not silently invent it.

### Tracing recovery

Tracing automatically reconnects its internal source-stream subscriptions after disconnect or crash using the existing 2× exponential backoff multiplier and full-jitter algorithm. After each unsuccessful attempt, the backoff cap doubles and is capped at the configured maximum; the actual delay is selected uniformly at random from zero up to that cap. The settings remain deployment-configurable through trace.reconnectInitialDelay and trace.reconnectMaxDelay, defaulting to 1 second and 60 seconds respectively. A successful reconnection resets the backoff.

Tracing is observational and may resume after a source stream gap by recording a diagnostic containing the affected source stream and missing sequence range. It does not fabricate missing source events or infer Security decisions. The gap diagnostic remains available on the tracing stream. A detected disconnect or gap degrades Tracing health; successful reconnection clears the degraded warning while retaining the diagnostic history. This does not weaken Security's separately settled strict continuity and fail-closed requirements.

### Trace access API

GET /v1/streams/{streamId}/trace remains the documented external trace access surface. It exposes Broker-owned trace messages for the requested tracing stream or stream scope; it is not an independent data source. External access uses the ordinary Application subscriber identity, approval, credential, stream-scope, revocation and rate-limiting rules. Internal Tracing uses its reserved System identity and the ordinary subscription mechanism. The API and built-in monitor must not bypass authentication or create a special trace-only authorisation path.

The exact relationship between the trace endpoint's streamId parameter, the tracing stream schema/identity, the trace event's source stream identifier and Broker's normal stream envelope must be made explicit in api/asyncapi.yaml before implementation. Do not imply that each source stream has an independently stored trace history unless that is deliberately represented as Broker-owned stream data.

## Stream monitor page

`/streams` serves an optional HTML trace viewer for PulsyFlux users who do not want to build their own monitoring interface. The page lists every stream and lets the user select one stream to view its live trace in chronological order. It is a browser client of the publicly reachable `GET /v1/streams/{streamId}/trace` SSE operation; that operation remains the sole trace-data interface and source of truth. External clients may consume the same API directly and are not required to use this page, but they must establish an SSE connection and obtain access through the ordinary external `Application` subscriber authorisation workflow. The requesting Subscriber ID identifies the Application subscriber; the authorisation scope is the requested Stream ID. Once approved for that stream, the client receives the complete stream trace, including retained history and subsequent live activity involving other subscribers and publishers, rather than a subscriber-specific trace. With no `Last-Event-ID`, the server replays from the earliest retained trace event; otherwise the existing trace resume and gap rules apply. The connection remains pending and no trace records are delivered until the standard Security pub/sub workflow approves access; the endpoint is not anonymous.

### Behaviour

#### Stream list

One card per stream, from `GET /v1/streams`.

Each card shows:

- The schema `title`.
- A shortened stream ID, with the full ID copyable.
- The counts.
- An activity sparkline.

The activity sparkline is derived client-side from the change in that stream's accepted publish count between successive `GET /v1/streams` polls. Rejected publish counts are not included in the sparkline.

The page refreshes the stream list by polling `GET /v1/streams` every 2 seconds. Newly registered streams then appear without a page reload.

The initial contract has no separate stream-list SSE operation.

#### Stream view
Selecting a card opens only that stream's trace from `GET /v1/streams/{streamId}/trace` as a vertical timeline, newest at the bottom, with auto-scroll that pauses when the user scrolls up. The page maintains one active trace SSE connection for the selected stream; selecting another stream switches the view and connection to that stream.

#### Controls

- Filters by event kind and subscription. When a subscription filter is active, the page shows events belonging to that subscription and retains the corresponding `publish.accepted` events as context when they share the same `runId` and message `seq` with a visible `delivery`. It hides unrelated publishes and deliveries from other subscriptions. Filtering changes presentation only; it does not change trace records or their `traceSeq` order.
- Pause closes the selected stream's trace SSE connection and retains the most recently received SSE event ID as the resume cursor. For trace events this is their `traceSeq`; for a `trace.gap` control event it is the event's `toTraceSeq` ID.
- Resume opens a new trace SSE connection for the selected stream using the retained cursor in the standard `Last-Event-ID` HTTP header.
- Events still present in the retained in-memory trace history after the cursor are replayed.
- Events evicted from the in-memory trace history after the cursor are reported through `trace.gap`, after which replay continues with retained events.
- Jump to `traceSeq`.

The page must not imply that trace events evicted from the bounded in-memory history can be recovered from persistence files during the same process run. Persistence files are not a live replay source.

Gaps are shown as gaps.

### Visual design

The page should be distinctive and pleasant to use, not a plain table.

Each stream gets a colour and a small generated pattern derived from its hash bytes.

The same stream always looks the same, and two streams are easy to tell apart.

The timeline uses two lanes:

- Publishes on the left.
- Deliveries on the right, grouped by subscription.

A thin connector links each `publish.accepted` to its `delivery` events, so you can see which subscribers a message reached and how long it took (the `at` difference).

Each kind has its own colour and icon:

- Rejected publishes, gaps and overflows stand out.
- Opened and closed subscriptions mark the start and end of that subscription's lane.

Light and dark themes follow `prefers-color-scheme`.

The layout works from laptop to wide screens.

Motion is subtle and turned off under `prefers-reduced-motion`.

### Rules

- The page is embedded static HTML, CSS and JavaScript with no build step and no third-party libraries. Its browser-side SSE handling is private implementation code for this built-in viewer, not a separately served script, public SDK or client library.
- Because Pause/Resume must explicitly set `Last-Event-ID` on a newly opened connection, the viewer uses fetch-based SSE consumption rather than relying on native `EventSource` for that reconnect. It still connects only to the public trace operation; no private trace endpoint or alternate trace protocol is introduced.
- It loads nothing remote and is covered by the same same-origin CSP header as the docs page.
- The page uses only contract operations: list streams, the trace operation and get schema.
- It has no private endpoints.
- `/` (docs) and `/streams` (monitor) link to each other.

## Authentication administration page

Pulsyflux provides an HTML administration page for reviewing and managing Application subscriber access. It is served locally by the existing Pulsyflux service.

### Behaviour

#### Authentication

The page authenticates as the single global Admin subscriber. The administrator enters the token configured through `PULSYFLUX_ADMIN_TOKEN`; the page presents it as a bearer credential and Pulsyflux validates it using the same token hashing, in-memory lookup and role resolution used for Application, System and Security subscribers.

There is no separate administrator login protocol, username, password, browser session or special authentication bypass. After the administrator enters the token, the page may use it transiently in browser runtime memory to make authenticated requests. It must not be embedded in the static page's HTML, CSS or JavaScript assets, placed in URLs, persisted in browser storage, or written to logs or traces. The token is not returned by any API.

Pulsyflux cannot start if `PULSYFLUX_ADMIN_TOKEN` is missing or empty, so every running instance has exactly one Admin credential record.

#### Stream selection

The page retrieves the available stream list through the public, read-only `GET /v1/streams` operation; stream discovery does not require the Admin credential. The administrator must still authenticate as the global Admin subscriber through the standard bearer-credential model before reviewing pending Application subscribers, approved relationships or performing administration operations. Selecting a stream changes the target stream; it does not change or reissue the Admin credential.

#### Pending and approved subscribers

Pending requests are held in memory and displayed with Subscriber ID and Name. The administrator can explicitly approve or reject each pending request.

Approval creates an in-memory Application credential record for that Subscriber ID + Stream ID relationship. It allows the associated pending SSE subscription to proceed, provided its waiting period has not expired. The credential is delivered once on that existing connection.

The page also allows the administrator to revoke an approved Application subscriber–stream relationship. Revocation immediately invalidates its credential and closes active connections for that relationship.

#### Role boundaries

- The Admin credential is global and there is exactly one Admin record.
- Application subscribers cannot assign themselves a role or invoke Admin operations.
- System and Security subscribers are created and managed internally by Pulsyflux.
- The Security subscriber uses the same subscription and publishing mechanisms as other subscribers. To prevent recursive authentication, the core recognises its trusted, startup-provisioned Security identity locally and does not delegate that identity's own role check back to the Security subscriber. This is a narrowly scoped trust check, not a separate subscription or publishing path.
- Admin can administer every stream; the selected stream scopes an individual approval or revocation operation.
- Ordinary stream approval never creates or modifies an Admin, System or Security credential.

### Visual design

The page should be clear and pleasant to use. The selected stream must be obvious. Pending subscribers and approved relationships should be easy to distinguish. Approval, rejection and revocation actions must be explicit and must not be triggered accidentally. Light and dark themes follow `prefers-color-scheme`. The layout works from laptop to wide screens.
### Rules

- The page is embedded static HTML, CSS and JavaScript with no build step and no third-party libraries.
- It loads nothing remote and is covered by the same same-origin CSP header as the docs page.
- The Admin bearer token may be used transiently in browser runtime memory after entry, but must not be embedded in the static page's HTML, CSS or JavaScript assets, placed in URLs, persisted in browser storage, or written to logs or traces.
- State-changing administration operations must reject cross-origin requests.
- All credential hashes and role records are in memory only and disappear on process exit.
- The page uses the existing Pulsyflux service and unified subscriber authentication component; it does not introduce a separate application or service.
- `/`, `/streams` and the authentication administration page link to each other where appropriate.

## Docs and "Try it" console

`/` serves the docs page.

`/asyncapi.yaml` serves the embedded contract bytes.

The page renders the contract with `@asyncapi/web-component`, vendored at a pinned version and copied unchanged into `internal/web/assets/`.

No CDN is used at runtime.

Below the rendered docs, the generated console lets a user:

- Register a schema and see the returned `streamId`.
- Publish a message, prefilled from the contract examples.
- Open a live SSE subscription and watch events and gaps arrive.
- Resume an SSE subscription using the standard `Last-Event-ID` HTTP header when the client environment supports setting it.

The console uses only the generated console data from `console.gen.go`.

A contract change updates it on regeneration.

## Package layout

```text
api/asyncapi.yaml                  contract (only source of truth)
cmd/pulsyflux/                     composition root only
cmd/contractgen/                   generator and check modes, embedded meta-schemas
generated/api/                     committed generated code
internal/streamid/                 schema admission, JCS, hashing
internal/broker/                   streams, seq, retention, replay, subscriber queues
internal/service/                  implements api.Handler using broker and streamid
internal/auth/                     internal authentication subscriber, in-memory pending approvals and subscriber–stream sessions, administrator-token validation, credentials, authorisation and revocation
internal/core/                     shared lifecycle, execution context, configuration, snapshot and persistence capabilities
internal/trace/                    Tracing messaging participant and trace-event interpretation; no independent stream queue, message history or persistence store
internal/web/                      docs page (/), stream monitor page (/streams), authentication administration page, vendored assets
internal/integration/              HTTP/SSE contract tests
internal/testsupport/              shared SSE test client and contract assertions
```

## Go conventions

These conventions incorporate the project-wide Go Coding Standards.

They apply to all hand-written Go code in Pulsyflux.

Where a general convention conflicts with an explicit Pulsyflux architecture or testing rule elsewhere in this file, the Pulsyflux-specific rule wins. Do not introduce unrelated infrastructure merely to imitate a pattern from another project.

### Package structure and naming

- All non-entry-point production code lives under `internal/`. `cmd/<name>/` contains entry points only; `cmd/pulsyflux` is the composition root and builds the complete object graph.
- Each package has one clear responsibility. Package names are short, lowercase, single words, with no underscores, and are named for the concern rather than a concrete type. If a name needs multiple words, reconsider the package boundary.
- Constructors are named `New(...)` and return the concrete type. Input parameter structs use the `Input` suffix where a named input type improves clarity (for example, `RegisterInput`).
- Exported identifiers use `PascalCase`; unexported identifiers use `camelCase`. Error types use the `XxxError` suffix. Do not use underscores in identifiers or source file names, except for blank identifiers and the required `_test.go` suffix.
- Test helpers use `newTestXxx(t *testing.T)` and call `t.Helper()` as their first statement.

### Dependency injection and composition

- Pass dependencies through constructors. Do not hide service instances in package-level variables, and do not use `init()` for business logic or object construction.
- The composition root in `cmd/pulsyflux` constructs and connects the complete object graph. Packages receive their dependencies; they do not reach into other packages' composition or global state.
- Define an interface only when at least two real implementations exist or a package boundary genuinely requires an abstraction to avoid an import dependency. Do not create interfaces solely to make tests substitute implementations.
- Put `var _ Interface = (*Type)(nil)` compile-time satisfaction checks immediately after the implementing type's definition (or at the implementation boundary when the interface is declared in another package). Keep the check in the same package and file as the implementation where practical. The generated `api.Handler` seam is required by the contract-first architecture.

### Errors

- Give every failure category that callers must distinguish a typed error with a stable semantic identity. Do not use a bare `errors.New` string for a condition callers need to branch on.
- Typed errors expose the information needed by callers; where an HTTP response is part of the public contract, map the typed error to the generated, documented response rather than duplicating response definitions in hand-written code.
- Wrap errors at package boundaries with operation-first context, using `%w`, for example `fmt.Errorf("append to stream: %w", err)`. Do not add empty or redundant prefixes such as `error:` and do not return an uncontextualised lower-level error across a boundary.
- Never panic for recoverable conditions. Panics are reserved for violated programmer invariants that cannot be represented as normal errors.

### Context and resource lifetime

- `context.Context` is the first parameter of every function that performs I/O, blocks, or supports cancellation. Never store a context in a struct.
- Pass `context.Background()` only at the process entry point. Propagate the caller's context through all other layers; derive and cancel child contexts for owned background work.
- Acquire and release resources in the same scope. Use `defer` for cleanup, including closing files and response bodies, unlocking mutexes, and calling cancellation functions.

### Concurrency

- Protect shared mutable state with `sync.Mutex` or `sync.RWMutex`, choosing based on the read/write pattern. Keep critical sections minimal and never hold a lock across I/O or while waiting for another goroutine.
- Every goroutine must have a defined lifetime and a way to stop: it completes its finite task or observes cancellation/context shutdown. No unowned, indefinitely running fire-and-forget goroutines.
- Background goroutines that cannot return errors to a caller report failures through the repository's structured logger (`log/slog`). Logging must not include message payload data.
- Follow the broker-specific channel ownership, fan-out, replay, and lock-ordering rules in Broker semantics. Those rules are more specific than these general concurrency conventions.

### Functions, closures and comments

- Prefer named functions and methods. Keep functions focused on one responsibility and return early on errors rather than building deeply nested conditionals.
- Do not put multi-statement business logic in anonymous callback functions. Extract it into a named function or method and pass required values explicitly rather than capturing surrounding state.
- Closures are permitted only for trivial deferred cleanup, a one-call `t.Cleanup`, standard-library sort/search callbacks, a goroutine launch that forwards to a named method, or an adapter such as `http.HandlerFunc` that directly references a named method. A closure with multi-statement logic should be extracted.
- Do not add explanatory comments throughout implementation code. Comments are permitted where required for Go tooling and documentation, especially doc comments immediately above exported identifiers; exported doc comments start with the identifier name. Prefer clear names and small functions over comments that narrate the code.
- Use short idiomatic variable names (`i`, `k`, `v`, `err`, `ok`) only in narrow scopes; use descriptive names for package-level declarations and parameters.
- Always check `scanner.Err()` after a `bufio.Scanner` loop.

### Imports and formatting

- Keep imports in three groups separated by blank lines: Go standard library, third-party modules, then this module's own packages.
- Alias an import only when needed to resolve a name collision or make a package's role unambiguous.
- Run `gofmt` on all Go source. Use a pinned `goimports` version if needed to enforce the required import grouping; do not let tooling silently reorder imports into a different grouping convention.

### Dead code

- Delete every production function, type, constant, variable or import that is not reachable or otherwise required from a production entry point. Re-check after removals because they can expose further dead code.
- A compile-time interface assertion (`var _ Interface = (*Type)(nil)`) is intentional compile-time code, not dead code.
- Test-only helpers are retained only while referenced by tests. Pulsyflux does not use shared test fakes or substitute implementations; see Testing.

### Go verification commands

After every Go change, run the complete verification set before committing:

```sh
go build ./...
go test ./...
go vet ./...
gofmt -l .
staticcheck ./...
```

All commands must succeed; `gofmt -l .` must print no files.

Tool versions, including the Go toolchain and `staticcheck`, are pinned by the repository/CI configuration rather than installed from an unpinned `@latest` during a build.

The Makefile build gates below may run these checks through their corresponding targets, but may not omit any of them.

## Makefile

### Build gates

#### `make generate`

Runs `go generate ./...`.

Deliberately regenerates committed generated output after a contract change.

Inspect and commit the resulting diff.

#### `make verify`

Runs:

- `contractgen -check-generated`
- `-check-format`
- `-check-architecture`
- `go mod tidy -diff`
- `go vet ./...`
- `gofmt -l .` (must be empty)
- `staticcheck ./...`
- `deadcode`

It must not regenerate or modify files.

#### `make test`

Runs `go test ./...`.

#### `make build`

Runs `verify`, then `test`, then builds `dist/pulsyflux`.#### `make package`

Builds the platform installer packages from the verified release binaries.

After changing `api/asyncapi.yaml`, run `make generate`, inspect and commit the generated output, then run `make build`.

CI runs `make build` on every pull request and push to `main`.

CI checks generated output without regenerating it, so stale committed output fails verification rather than being silently repaired.

The tag-triggered release workflow runs `make build` and `make package` on the release commit, validates the installers, and publishes the GitHub Release only after all required platform jobs pass.

Tool versions are pinned.

No gate is skipped, made warning-only or set to `continue-on-error`.

Run the full `go test ./...`.

Do not use subtest path filters.

## Development iteration tracking

Development is organised into small, verifiable iterations once implementation has been explicitly authorised.

Git history shows exactly what changed.

The iteration log provides a durable project-level record of progress and decisions. Do not keep an ever-growing chronological diary inside this steering file.

This file defines the tracking rules. Store implementation iteration records in the repository under `docs/iterations/` when implementation begins.

### Current project phase

The project is currently in the steering and design phase.

- Work is limited to reading, discussing, refining and updating project documentation and decisions, especially this `steering.md`.
- Do not implement application code, create the AsyncAPI contract, generate code, scaffold packages, add implementation tests, or begin packaging.
- Do not begin implementation until the user explicitly says that the steering/design phase is complete and authorises implementation.
- A proposed design is not an approved decision until it has been agreed with the user and recorded in this file.
- When an unresolved decision affects architecture or externally visible behaviour, resolve it here before implementation.

### Iteration unit and workflow

After implementation is explicitly authorised:

- An iteration is a small, coherent increment that can be implemented, verified and reviewed as one unit.
- Work directly on the `main` branch. Do not create, use or switch to other branches.
- Do not create GitHub Issues or pull requests. Track goals, scope, decisions and outcomes in the iteration records and commit history.
- Each iteration must have a clear goal and observable completion criteria. Prefer a working vertical slice over a large phase of disconnected implementation.
- If an iteration is split, blocked, descoped or its acceptance criteria change, record that explicitly in its iteration record. Do not silently redefine completion.
- Keep commits focused and describe the purpose and scope accurately.

### Iteration records

Create one Markdown record per implementation iteration in `docs/iterations/`, named `NNN-short-description.md` (for example, `001-project-scaffold.md`).

Maintain `docs/iterations/README.md` as a concise index in iteration order, linking each record and showing its status.

Each iteration record contains:

1. **Identity:** iteration number, title, branch (`main`), and relevant milestone or release target if any.
2. **Goal:** the user-visible or engineering outcome intended, and why it matters to Pulsyflux.
3. **Scope:** what is included and explicitly excluded from this iteration.
4. **Requirements:** the requirement IDs or acceptance criteria being addressed. If a requirement is not yet written, link the relevant steering decision instead; do not invent requirements.
5. **Plan:** the small implementation steps and expected deliverables.
6. **Verification:** the exact commands and checks to run, plus any platform-specific manual checks. Record which checks actually ran and their results; distinguish not run, failed and passed.
7. **Outcome:** what was delivered and the resulting commit(s) or relevant artifacts.
8. **Deviations and decisions:** scope changes, trade-offs, newly discovered constraints and any steering/spec changes required.
9. **Follow-ups:** unresolved defects or work intentionally deferred, recorded in the iteration log.

### Status and completion

Use only these statuses in the index and iteration record:

- `planned`
- `in progress`
- `blocked`
- `completed`
- `cancelled`

Definitions:

- `planned`: agreed and not started.
- `in progress`: implementation or verification has started.
- `blocked`: progress cannot continue until a named dependency or decision is resolved; state what is needed.
- `completed`: acceptance criteria passed and verification results recorded.
- `cancelled`: intentionally abandoned; record why and whether replacement work exists.

Update the index and iteration record in the same commit that changes the iteration's status or materially changes its scope.

Keep records factual and concise.

Never claim a test, installer, service lifecycle check or release step passed unless its result was observed and recorded.

The iteration log supplements, but does not replace, commit history, the requirements/specification documents or CI results.

It must not become a second source of product decisions: decisions that change scope or externally visible behaviour must be reflected in this steering file or the owning specification, according to the authority order above.

## Git workflow and GitHub releases

GitHub is the source-of-truth remote for the repository, commits, CI and published releases.

The release workflow produces installable packages for Windows and Linux; a release is not considered complete if it contains only raw build binaries.

### Branch and change workflow

- All repository work is performed directly on the `main` branch.
- Do not create, use, check out, merge or push any other branch.
- Do not create GitHub Issues or pull requests. Project planning, open decisions and progress are recorded in `steering.md`, implementation iteration records and commit history.
- Before changing a file, inspect the current `main` version and preserve unrelated content.
- Keep commits focused. A contract change, its generated output, implementation and tests belong in the same logical change once implementation is authorised.
- Keep `main` buildable and passing the full verification and test gates once implementation begins.
- A change to `api/asyncapi.yaml` must include the regenerated `generated/` output. CI fails if generated output is stale.
- No gate is skipped or made advisory.

### Versioning and release trigger

- Use semantic version tags in the form `vMAJOR.MINOR.PATCH` (for example, `v1.2.3`).
- A GitHub Release is created from a version tag. The tag must point to a commit on `main` that has passed the required CI workflow.
- The release workflow validates that the tag is a valid semantic version and that the version embedded in release metadata matches the tag.
- Release artifacts are built from the tagged commit, not from an uncommitted workspace or a later branch head.
- Releases are published as GitHub Releases with generated release notes and attached installer packages.

### Installer artifacts

The first release targets 64-bit Windows and Linux on `amd64` and `arm64` where the packaging tools support the target.

Each package installs:

- The `pulsyflux` executable.
- The operating-system service definition.
- The files required for local use, including the default configuration template and bundled documentation/assets where applicable.

The installer configures Pulsyflux as a managed background service and starts it after installation.

The service remains local-only: it listens only on a loopback address and makes no outbound runtime network connections.

#### Windows
Produce a signed-ready installer executable (`.exe`) using Inno Setup.

The installer:

- Requires elevation for machine-wide installation.
- Registers Pulsyflux with the Windows Service Control Manager.
- Configures automatic startup.
- Starts the service after installation.
- Runs the service under the unprivileged `NT AUTHORITY\LocalService` account.
- Does not run it as `LocalSystem` or as an interactive logged-in user.
- Grants that account write access only to Pulsyflux's required configuration, state and trace directories.- Supports install, upgrade and uninstall.

Upgrades stop the service, replace application binaries and service metadata, preserve configuration and state, then restart the service.

Uninstall stops and removes the service and application binaries but preserves configuration and state by default.

Offer explicit removal only as a separate user choice.

#### Linux

Produce `.deb` and `.rpm` packages for supported architectures.

Install a `systemd` unit for `pulsyflux.service`.

Create a dedicated unprivileged system user and group named `pulsyflux` with no interactive login.

Grant it access only to the required configuration, state and trace directories.

Enable the unit for automatic startup at boot and start it after installation.

Configure systemd to restart the service after an unexpected failure, while allowing an intentional stop or package removal to remain stopped.

Package upgrades stop the service, replace application binaries and unit metadata, preserve configuration and state, reload systemd as needed, and restart the service.

Ordinary package removal stops and disables the service and removes package-owned binaries and unit files, but preserves configuration and state.

Purge/removal of persistent data must be an explicit separate action.

#### Both platforms

- Both service integrations must provide graceful shutdown.
- Both must use the same production bootstrap and configuration as a foreground development run.
- All runtime paths must be explicit and writable by the service identity.
- The service must not depend on a user's home directory, terminal, interactive session, or current working directory.
- Installation and service startup must not require internet access.
- Provide documented service-management commands for status, start, stop and restart using the native operating-system service manager.
- Do not add a second custom service supervisor or a separate application-level daemon protocol.
- Package metadata includes the product name, version, architecture, publisher/project URL and license.
- The license file and release notes are included in the package or linked from its metadata as appropriate.
- The installers do not bundle Go, require an internet connection, download runtime dependencies, or install a database.
- Runtime assets are embedded or installed locally.
- The release includes SHA-256 checksums for every installer and a machine-readable checksum manifest.
- Checksums are generated after packaging and verified before publication.
- Build and packaging steps run in GitHub Actions on the appropriate Windows and Linux runners.
- Pin action versions and packaging tool versions.
- Do not store signing keys or credentials in the repository.
- Signing can be enabled only through protected GitHub secrets and must not block unsigned development builds.

### Release verification

Before publishing, the release workflow must:

1. Check out the exact tagged commit and verify the tag/version.
2. Run the full `make build` gate.
3. Build the Windows installer and Linux `.deb` and `.rpm` packages for each supported architecture.
4. Run package-level checks on native Windows and Linux runners:
   - Verify package metadata and contents.
   - Install into a clean test environment.
   - Verify the Windows service or Linux `systemd` unit is registered, enabled for automatic startup and running.
   - Verify the service starts after a reboot or equivalent service-manager restart.
   - Confirm it binds only to loopback.
   - Exercise stop and restart.
   - Uninstall/remove the package and verify the service is stopped and removed while configuration and state are preserved by default.
5. Generate and verify SHA-256 checksums for all release artifacts.
6. Attach installers, checksums and release notes to the GitHub Release.

Publish only after all required jobs succeed.

The release workflow must be reproducible from a tag: rerunning it for the same tag must use the same source commit and produce the same set of named artifacts.

Where byte-for-byte reproducibility is not achievable because of platform packaging metadata, record that limitation. Do not silently claim reproducible binaries.

## Open decisions

### Settled foundational architecture — Broker streams are the only component communication model

The user has clarified and settled the macro-architecture: Application, Security, Tracing and Admin are independent participants in one shared Broker infrastructure. Each can publish to and subscribe to streams according to its responsibility, authenticated identity, role and scope. They are not merely subscribers; each may be both publisher and subscriber. Components communicate with one another through Broker streams, not direct method calls, RPC, private channels or a separate component-to-component abstraction. The publisher does not need to know its subscribers, and subscribers do not need to know publisher implementations. Broker owns streams, authoritative message queues/logs, published messages, ordering, fan-out, routing, dispatch, subscriptions, delivery, retention and replay. Per-subscriber queues are bounded delivery buffers, not separate message histories.

Application is the client-facing/external application participant. Security and Admin are internally provisioned participants. Tracing participates internally and also has an external trace access/integration surface; both use the same Broker-owned stream data and existing authentication/authorisation model. Each participant's behaviour differs by domain responsibility, not by using separate messaging infrastructure.

The user has also clarified the relationship between Broker and Core: Broker owns its queues and published messages. Stream snapshots and persistence operate on that Broker-owned stream/queue/message state; Core provides the shared reusable snapshot/persistence mechanism and policy. There must be no duplicate authoritative stream store and no subsystem-specific queue/history/persistence implementation for messages already owned by Broker. Persistence restoration restores Broker-owned stream state before it is used. Existing stream fan-out semantics remain in force: every authorised subscriber receives its own delivery; this is not competing-consumer work-queue semantics.

This decision supersedes any description of direct component-to-component messaging, independent subsystem message buses, Tracing-owned stream history/queues, or Tracing-owned message persistence. Preserve domain-specific trace event schemas, authentication rules, delivery guarantees and correctness-specific gap handling, but express them through the common Broker stream model. Any remaining conflict between the new ownership model and older trace-specific cursor, sequence, file, snapshot or replay rules must be reconciled before implementation; do not silently preserve a second source of truth or invent a mapping.


The former “Internal subscribers and the Security role” discussion has been reconciled with the authoritative settled decisions below. Internal subscriber identity and provisioning, the existing Security authorisation request/decision contracts and validation, delivery and failure behaviour, timeout handling, tombstone retention, and recovery are settled; they are not open design topics.

The universal HTTP rate-limiting requirement, token-bucket mechanism, four endpoint categories and their default refill rates and bucket capacities are settled: public read/discovery 120 requests/minute with capacity 30; write/mutation 60 requests/minute with capacity 15; SSE establishment 60 requests/minute with capacity 10; and administration 30 requests/minute with capacity 10. Each category remains independently configurable; new endpoints use an existing category unless an explicit decision adds another. Identity/IP accounting, bounded bursts, standard `429` responses and trusted-proxy-only forwarded IP handling are also settled. No further rate-limit default decision is open. HTTP request rate limiting is independent of the already-settled SSE concurrent-connection limit. The trace-retention scope is settled: each stream has one shared retained trace history visible to every authorised subscriber; subscriber-specific trace retention is not used. The trace-cursor-below-retained-history behaviour described in Settled decisions was explicitly agreed and is no longer an open topic.

The trace-persistence snapshot commit protocol is settled: complete snapshots are written and synced to a same-directory temporary file and atomically replace the active file; backup policy and the interaction with `trace.maxFileBytes`/`trace.maxFiles` remain open. Restoration validation/reconciliation, persistence retry/terminal-failure policy, exact in-memory retention capacity, and shutdown deadline/behaviour when storage remains unavailable also remain open.

The central-Core stream persistence direction and its foundational scope are settled: persistence and snapshots are a shared Core capability reusable across pub/sub modules for `Application`, `System`, `Security` and `Admin`. The stream class is determined by the stream's role. Persistence policies are defined at stream-class level, with optional per-stream overrides for exceptions. A stream class with no explicit persistence policy is non-persistent by default: persistence requires an explicit class-level opt-in, while a per-stream override may opt an individual stream in or out. This decision does not make authentication/authorisation state, credentials, pending requests, sessions, or all Security/Admin control-plane events persistent; the existing in-memory-only authentication and authorisation state remains unchanged. Assess sensitive data, security boundaries, retention, replay and recovery when reviewing subsequent policy details. No implementation.

Any newly identified decision that changes scope, externally visible behaviour or an acceptance criterion must be discussed and explicitly agreed before it is added to **Settled decisions**. Do not infer or silently settle decisions.

## Next-session handoff

**Latest architectural decision:** Application, Security, Tracing and Admin all participate in one shared Broker infrastructure and may both publish and subscribe to streams. Component-to-component communication is exclusively through Broker streams; there is no direct-call/RPC/private-channel messaging architecture. Application is the client-facing/external participant; Security and Admin are internally provisioned; Tracing participates internally and also has an external trace access/integration surface. Broker owns streams, authoritative queues/logs and published messages, ordering, fan-out, routing, dispatch, subscriptions, delivery, retention and replay. Core supplies shared snapshot/persistence capabilities and policy operating on Broker-owned stream state; no duplicate subsystem message stores are allowed. Reconcile older Tracing-specific history, queue, snapshot, persistence, cursor and replay rules with this ownership model before implementation, preserving domain-specific trace contracts and correctness rules where compatible.

**Resume from the latest `main` branch of this file.** Before proposing anything, read this handoff, the complete **Settled decisions** section, and the **Open decisions** section. Continue architecture discussion only: do not implement, scaffold, or create code. Discuss exactly one architectural decision at a time. For each proposal, explain both subsystem-specific behaviour and macro-architectural implications. A proposal explicitly marked NOT settled must remain unsettled until the user agrees. After each agreement, update this file directly on `main`, preserve all prior decisions, and verify the committed content by fetching it again.

**Newly settled Core snapshot/persistence model:** Every pub/sub stream has a complete in-memory snapshot governed by its JSON schema. The snapshot always exists in memory; persistence policy only selects which schema-defined properties are written to durable storage. A stream's role determines its stream class. Persistence policies are class-level with optional per-stream overrides, and default to non-persistent unless explicitly opted in. Pub/sub subsystems must not create independent files, memory stores or duplicate stores for stream state already represented by Core snapshots. Tracing and Security use the shared Core model. Security identities, roles, credential hashes, pending approvals and authorisation state are represented in the in-memory snapshot, but Security snapshots are not persisted; plaintext bearer tokens are not retained in snapshots. Authentication/authorisation state remains non-persistent. This architectural direction supersedes conflicting subsystem-specific storage rules elsewhere in this document; review and remove/reframe those obsolete rules rather than preserving duplicate storage architectures. Next review: reconcile the remaining Tracing-specific file/history/worker/rotation/recovery rules and other subsystem-owned stores against this model, one coherent section at a time. Read the complete **Settled decisions** and **Open decisions** before continuing. No implementation.

**Last completed discussions:** The user agreed to the memory-first tracing foundation: ordinary in-memory pub/sub feeds a bounded per-stream in-memory trace history that is the live source of truth for replay, retention, gap calculations and SSE; asynchronous local JSON Lines persistence exists only for restart restoration and cannot block live operation. The configurable persistence flush target is 50ms by default; an unexpected crash or power loss may lose events since the last successful flush. This crash-loss window is accepted. The user then agreed to coalesced persistence scheduling and atomic snapshot writing: each stream has a dirty indication rather than an event-per-event persistence queue; multiple updates collapse, and the worker snapshots the latest retained history. Each pass writes a complete JSON Lines snapshot to a same-directory temporary file, syncs and closes it, then atomically replaces the active file without deleting the previous committed snapshot first. Incomplete temporary files are not committed snapshots; backups, where configured, are preserved before replacement and pruned only after commit. Directory metadata is synced where supported. Restoration integrity/reconciliation, backup policy, the interaction with file-size/rotation settings, and precise shutdown handling remain open for one-at-a-time review. The user also agreed that Pause closes the selected stream's trace SSE connection and retains the most recently received SSE event ID as the resume cursor. Resume reconnects with that cursor in `Last-Event-ID`, relying on existing replay and `trace.gap` behaviour after in-memory retention eviction. The cursor includes the `toTraceSeq` ID when the last received event is a `trace.gap`. The user also agreed to the precise retention-eviction `stream.gap` boundary rule. For a valid current-run cursor at sequence `seq` and oldest retained sequence `oldestSeq`, emit exactly one inclusive gap from `seq + 1` through `oldestSeq - 1` only when `seq + 1 < oldestSeq`, then replay from `oldestSeq`. If `seq + 1 == oldestSeq`, replay immediately without a gap. Gaps must never be empty, overlap, or include retained messages. This rule applies uniformly to Security, Tracing and all other subscribers; their already-settled recovery responses remain distinct. The user has also agreed that all authorisation event JSON fields use camelCase: `correlationId`, `subscriberId` and `streamId`; that administrator approval/rejection instructions must enter Security's existing event-driven workflow through the dedicated internal authorisation stream; that the instruction event is named `authorization.administratorDecisionRequested`; and that the instruction payload contains exactly `correlationId`, `subscriberId`, `streamId` and `requestedDecision`, with `requestedDecision` exactly `Approved` or `Denied`. The correlation ID is the original core-generated request UUID; the operation is not repeated. The user has now also agreed to the administrator instruction authentication and structural validation contract: only the authenticated global Admin principal may submit; the core validates the exact four-field payload, UUID identifiers and decision enum before publication; invalid or unauthorised instructions are not published; Security remains the decision-maker and the core applies its validated decision. The user has also agreed to duplicate and stale administrator instruction handling: repeated instructions may produce repeated decisions, but the core's existing first-valid-decision rule prevents repeated or conflicting decisions from changing authorisation more than once; expired, unknown and already-acted-on correlations cannot create or revive requests or change permissions; no separate instruction deduplication registry, acknowledgement, retry or tombstone policy is added. The user clarified the 10-minute Admin-mediated Application approval window: an unapproved pending request expires after 10 minutes and is no longer eligible for approval; expiry is not an explicit denial. The ordinary 5-second Security response timeout does not expire this Admin-mediated request. If approval is accepted before the 10-minute window ends, later SSE inactivity/connection timeout does not reverse, revoke or otherwise change the established authorisation. These decisions are recorded in the authoritative sections below.

**Other recent decisions that must remain settled:**
- Security becomes unavailable immediately on subscription detachment, including queue overflow. It can resume only after replay proves uninterrupted continuity from its last successfully processed event position. An unrecoverable `stream.gap` permanently disables Security for the remainder of that process run; recovery requires restart. The core remains fail-closed under the previously agreed existing-connection policy.
- Internal subscribers use a distinct tagged internal-principal identity namespace, separate from Application Subscriber ID UUIDs. Reserved identities such as `security` and `tracing` are established only through trusted startup provisioning and credential records; this does not create a separate authentication or messaging path or grant blanket privileges.
- Tracing is observational and uses the ordinary in-memory pub/sub path. Its bounded per-stream in-memory history is the live source of truth; independent asynchronous file persistence exists only for restart restoration. Persistence scheduling uses a per-stream coalesced dirty indication, not an event-per-event writer queue; updates may collapse and the worker snapshots the latest retained history. Persistence failure or backpressure never drops live trace events or blocks live operation. The earlier file-backed live-tail/SSE architecture and any rule that disables live tracing because of a file failure are superseded. Restoration validation/reconciliation and precise shutdown handling remain open. Backup policy and the interaction between complete snapshots and `trace.maxFileBytes`/`trace.maxFiles` remain open.
- Security's event-driven authorisation workflow, validation, deadlines, fail-closed handling, bounded workflow-owned tombstones, and recovery defaults are settled as recorded below. Preserve the distinction between Security's strict continuity requirement and Tracing's automatic warning clear after successful reconnection while retaining its gap diagnostic.

**Superseded trace-file failure decision:** The earlier decision to permanently disable an affected stream's live trace recorder when live file tailing, startup file validation, or file writing/rotation fails applied to the former file-backed live architecture. It does not apply to live in-memory tracing under the newly agreed foundation. A file failure may degrade persistence and restart durability, but must not disable the in-memory trace history, live trace SSE, or Tracing's observation path. The persistence dirty indication remains pending through write failure so later successful persistence can converge to current retained history; retry timing and detailed recovery policy remain open. Do not infer repair, truncation, overwrite or sequence-reconciliation behaviour from the superseded file-sink rules.

**Newly settled foundational decision — memory-first tracing:** The user agreed that Tracing observes through ordinary in-memory pub/sub; each stream's bounded in-memory trace history is the live source of truth for ordering, retention, replay, gap calculations and SSE; trace SSE never reads/tails files; and a separate asynchronous persistence layer saves retained history to local JSON Lines files for restoration after restart. Persistence must not block publishing, delivery, in-memory recording or SSE reads. Persistence failure affects restart durability only and is reported separately by `/health`; live in-memory tracing remains operational. The persistence flush target is 50ms by default and configurable. An unexpected process crash or power loss may lose trace events accumulated since the last successful flush; this crash-loss window is accepted and affects restart restoration only. Coalesced persistence scheduling and atomic snapshot writing are settled. Each persistence pass writes a complete JSON Lines snapshot to a same-directory temporary file, flushes and syncs it, closes it, then atomically replaces the active file without deleting the previous committed snapshot first. Incomplete temporary files are not committed snapshots; configured backups are preserved before replacement and pruned only after commit; directory metadata is synced where supported. The atomic replacement is the commit point, and no manifest or second persistence index is introduced. Restoration validation/reconciliation, backup policy, the interaction with `trace.maxFileBytes`/`trace.maxFiles`, and precise shutdown handling remain unresolved and must be reviewed one at a time. Do not implement anything. Continue the full-document consistency review one architectural decision at a time, explaining subsystem-specific behaviour and macro-architectural implications. The trace-retention scope decision remains settled: each stream has one shared retained history for all authorised subscribers. Do not reopen it or other completed trace decisions above. The user has settled that stream listing (`GET /v1/streams`) and schema retrieval (`GET /v1/streams/{streamId}/schema`) are public, read-only discovery operations requiring no subscriber authentication. Every HTTP API endpoint uses standard token-bucket rate limiting. Requests are accounted by authenticated principal after credential verification, otherwise by client IP; pending authorisation SSE establishment remains IP-accounted until a credential is later presented. The four endpoint categories and default refill rates/bucket capacities are settled: public read/discovery 120 requests/minute, capacity 30; write/mutation 60 requests/minute, capacity 15; SSE establishment 60 requests/minute, capacity 10; and administration 30 requests/minute, capacity 10. Each category's rate and capacity are independently configurable; new endpoints are assigned to an existing category unless an explicit decision adds another. These operational defaults protect the loopback-only service against accidental request loops and resource exhaustion, not as security boundaries. Excess requests receive `429 Too Many Requests` with `Retry-After` where practical, and forwarded client-IP headers are trusted only from explicitly configured proxies. HTTP request rate limiting is distinct from the existing SSE concurrent-connection limit: it applies to the HTTP request establishing an SSE connection, not events sent over an established connection. Preserve the single shared authentication and authorisation model; do not introduce another mechanism, privileged bypass or internal-role exemption. Read this handoff, the complete **Settled decisions** section and **Open decisions** before continuing. Architecture/design only: no implementation, scaffolding or code changes without explicit authorisation. Discuss exactly one architectural decision at a time and verify any steering changes after committing directly to `main`.

**Most recent completed review decision:** The user selected shared trace retention per stream (Option A). Each stream has one retained trace history, and all authorised subscribers to that stream use the same history, rotation/eviction boundaries and replay/gap calculations. There are no subscriber-specific trace retention windows. This is recorded in the Tracing section and Settled decisions below.

**Most recent completed trace-review decisions:** The user confirmed that `GET /v1/streams/{streamId}/trace` is a publicly reachable SSE interface, but trace data is not anonymous: external clients establish the SSE connection and use the ordinary external `Application` subscriber Security pub/sub authorisation workflow. The internal Tracing `System` subscriber also uses the standard Security workflow and receives no exemption. Authorisation is scoped to the requested Stream ID; an approved Application subscriber sees the complete trace for that stream, including activity involving other subscribers and publishers, for cross-subscriber debugging. Visibility includes retained historical trace events from before authorisation as well as subsequent live events. With no `Last-Event-ID`, replay begins at the earliest retained trace event; with a cursor, existing trace resume, replay and `trace.gap` rules apply. The connection remains pending and emits no trace records until Security approves access. These decisions are recorded in the Tracing and Stream monitor sections and are settled.

**Recent completed review decisions:** The user agreed that a current-run message cursor ahead of the latest assigned sequence is invalid and causes the broker to replay all retained messages; a malformed message cursor has the same fallback, with no `stream.gap`. Subscription registration and replay-snapshot capture are atomic under the stream lock; the subscription owner delivers replay in `seq` order before queued live messages, using the existing bounded subscriber queue and overflow handling without holding the stream lock during sends. A malformed trace cursor replays all retained trace events from the earliest available event without emitting `trace.gap`. Earlier settled monitor Pause/resume semantics, trace future-cursor behaviour and message retention-gap boundaries remain unchanged. Trace cursor continuity across process restarts depends on successful restoration of the prior sequence history from retained trace files. If no prior history is restored, a new history starts at `traceSeq` 1; an old cursor may be ahead of the new history and follows the existing future-cursor rule. Restoration and reconciliation details remain open.

**Latest verified review commit before this handoff update:** `c624fa972fb8dd7639c8988a81105316ddbb9f23` (`Include retained trace history after authorisation`). Other recent commits: `84025c80e2a9c75e01304ec214afca3c98a891c3` (ordered broker replay-to-live handoff), `86d04187bed7efb5a1af1e5718e97898e3b4cb35` (malformed message resume cursor), and `f3521ea6c5160cf0915cc7e8565a000f5c973b6c` (future message resume cursor).

No implementation or scaffolding is authorised.
## Settled decisions

The following points are settled by this steering file and are not open decisions:

- First-release stream and message storage is in-memory and is lost on process exit.
- Core owns a shared snapshot model for every pub/sub stream. Each stream's complete snapshot is always maintained in memory according to its JSON schema, irrespective of persistence.
- A stream's role determines its stream class. Persistence policy is defined at stream-class level, with optional per-stream overrides. The default is non-persistent; persistence must be explicitly opted in.
- Persistence policy selects which properties defined by the stream's JSON schema are written to durable storage. It does not determine which properties exist in the live in-memory snapshot.
- Pub/sub subsystems do not maintain independent files, memory stores or duplicate stores for stream state already represented by Core snapshots. Tracing and Security use the shared snapshot and persistence model rather than subsystem-specific storage architectures.
- Security state, including identities, roles, credential hashes, pending approvals and authorisation state, belongs in the shared in-memory snapshot model. Security snapshots are not persisted; sensitive/transient schema properties are excluded by persistence policy. Plaintext bearer tokens are not retained in snapshots; credential hashes are used for verification. Authentication/authorisation state remains non-persistent. 
- All authentication and authorisation state is held in memory only; no authentication files are created or read.
- Authentication uses one shared in-memory credential store and one bearer-token verification mechanism for all subscriber roles.
- Every HTTP API endpoint uses standard HTTP rate limiting, including public read-only endpoints, health and SSE connection-establishment requests. Use a standard token-bucket mechanism: consume tokens per request and replenish them at the configured rate up to the configured bucket capacity, allowing a bounded burst. Before core credential verification establishes an authenticated principal, account requests by client IP; once valid credentials establish the principal through the shared credential model, account subsequent requests by that principal. A new subscriber's pending authorisation SSE connection remains accounted by client IP until a credential is issued and presented on a later request; a client-supplied Subscriber ID is not an authenticated identity. The limiter observes identity established by existing authentication and must not verify credentials, make authorisation decisions or influence Security's decisions. Endpoint categories may have different configurable thresholds and normal burst handling. Excess requests receive `429 Too Many Requests`, with `Retry-After` where practical. Forwarded client-IP headers are trusted only from explicitly configured proxies. HTTP request rate limiting is separate from SSE concurrent-connection limiting and does not count events on an established SSE connection. The token-bucket algorithm, four endpoint categories and their default refill rates and bucket capacities are settled: public read/discovery 120 requests/minute with capacity 30; write/mutation 60 requests/minute with capacity 15; SSE establishment 60 requests/minute with capacity 10; and administration 30 requests/minute with capacity 10. Each category's rate and capacity remain independently configurable. These operational defaults protect the loopback-only service against accidental request loops and resource exhaustion; they are not security boundaries.
- The four roles are `Application`, `System`, `Security` and `Admin`.
- Application subscribers are external subscribers approved independently for each Subscriber ID + Stream ID relationship.
- System subscribers are Pulsyflux-managed internal subscribers to a stream; external clients cannot claim the System role.
- Security is a reserved internal role for the authentication and authorisation subscriber. Its identity and credential are provisioned by Pulsyflux at startup; external clients cannot request or claim it.
- Internal subscribers use a distinct, tagged internal-principal identity namespace separate from Application Subscriber ID UUIDs. Pulsyflux assigns reserved internal identifiers (such as `security` and `tracing`) and establishes them only through trusted startup provisioning and credential records; clients cannot claim an internal identifier or role. The shared credential record binds each authenticated principal to its provisioned role and stream scope, which the core enforces. Internal subscribers retain distinct identities while using the ordinary subscription and publishing mechanisms; this does not create a separate authentication mechanism, messaging path or blanket privilege grant.
- The Security subscriber uses the same subscription and publishing mechanisms as other subscribers. At startup, the core generates its 256-bit bearer credential, stores only its hash and trusted identity/role/scope record in the shared in-memory credential store, and hands the plaintext credential to the subscriber through a private, one-time in-memory handoff. The subscriber then uses ordinary bearer-token verification, subscription and publishing. The core's generic verifier resolves this pre-seeded Security record locally, so Security is not asked to authorise its own authentication or subscription and no separate messaging path is created.
- Bounded tombstone handling is a shared architectural pattern, not a mandatory central service or global registry. Workflows that need late-versus-unknown classification remove expired operations from active pending state and may retain bounded, metadata-only correlation tombstones; tombstones never change the outcome of an expired operation. Each workflow owns its implementation, lifecycle, retention and capacity configuration, chosen for its own deadlines, delivery behaviour and resource constraints. Reconsider a shared implementation only when multiple workflows demonstrate genuinely common requirements.
- When an authorisation request expires, the core removes it from the pending-request registry and retains a bounded, metadata-only tombstone for its correlation ID in memory. A subsequent decision matching an expiry tombstone is classified as late; a decision matching neither a pending request nor a retained tombstone is classified as unknown. When the core accepts the first valid decision for a pending correlation, it removes the request from the pending registry and retains a completed-correlation tombstone in the same bounded Security-workflow tombstone mechanism. A subsequent decision matching a completed-correlation tombstone is classified as duplicate, including a repeated or conflicting decision; it cannot change subscriber or authorisation state. Both expiry and completed-correlation tombstones are retained for 5 minutes from the corresponding expiry or successful decision, with a default maximum of 10,000 retained tombstones per running instance across both classifications. The 5-minute retention and 10,000-record capacity are specific to the Security workflow and deployment-configurable. When the capacity limit is reached, the oldest tombstone is evicted first; a later decision for an evicted correlation is classified as unknown. Tombstones never authorise or otherwise change permissions, contain no credentials, hashes or message content, and are cleared on process restart. Tracing records late, duplicate and unknown discard metadata independently under its existing trace retention and sink-failure behaviour; Security/core classification must not depend on Tracing availability.
- The Security subscriber owns subscriber-access and authorisation policy decisions: it receives relevant workflow events and decides whether Application access is pending, approved, rejected or revoked, and determines the permitted role and stream scope. The core owns foundational credential verification and enforcement: it verifies bearer credentials against the shared in-memory store, enforces the identity, role and scope on operations, manages pending and active SSE connections, enforces timeouts and connection limits, and performs the minimal bootstrap trust check for Security itself. The core does not delegate Security's own authentication or subscription authorisation back to Security.
- Security subscribes to events for new subscription attempts, reconnection attempts, Application approval and rejection outcomes, credential revocations, and subscriber identity or permission changes. These events provide Security with the workflow context needed for its policy decisions. Event payloads must not contain bearer credentials, credential hashes or message content; event schemas, ordering, delivery behaviour, failure handling, and late/duplicate/unknown decision handling are settled below.
- Tracing subscribes to both `authorization.requested` and `authorization.decided` on the shared internal authorisation stream and correlates them by `correlationId`, associating each decision with its original request and operation. It records every valid `Approved` and `Denied` outcome, plus late, duplicate and unknown decisions. Outcome records include event type, correlation ID, timestamp, subscriber ID, stream ID and decision, with the correlated request's operation where available. Discard records include event type, correlation ID, timestamp and discard reason. Tracing never records bearer credentials, credential hashes or message payloads. It is observational only, cannot approve, deny or alter authorisation, and its unavailability must not prevent Security or the core from processing authorisation.
- Trace retention is shared per stream, not per subscriber. Each stream has one bounded in-memory retained trace history, and every authorised subscriber to that stream replays from and observes the same retained history and the same retention-eviction boundaries; subscriber identity or connection lifetime does not create a separate retention window. The exact in-memory retention capacity/configuration remains to be specified. `trace.maxFileBytes` and `trace.maxFiles` govern persistence-file sizing and rotation only.
- Trace `traceSeq` continuity across process restarts depends on successfully restoring the prior retained history from persistence. If no prior history is restored, a new history starts at sequence 1; an old `Last-Event-ID` may then be ahead of the new history and is handled by the existing future-cursor rule, which waits for events with higher sequence values. No separate history-generation cursor is currently specified; restoration and reconciliation details remain open.
- Each stream's in-memory recorder/history and asynchronous persistence worker are created when its schema is registered and remain independent of subscriber activity. Persistence is scheduled through one coalesced dirty indication per stream; the worker snapshots the latest retained history rather than draining an event queue. Orderly shutdown should give pending persistence work an opportunity to flush before files close, but the shutdown deadline and behaviour when storage remains unavailable are not yet settled. On restart, history restoration occurs when the stream is registered; successful sequence continuation depends on the still-open restoration validation/reconciliation policy.
- Tracing's internal System subscriber automatically reconnects after disconnect/crash using its reserved startup-provisioned identity and credential, through the ordinary subscription mechanism and normal SSE replay. It receives no special broker privileges or unlimited queue, and its recovery never blocks Security or the core.
- An unrecoverable `stream.gap` is recorded in trace history with the affected source stream and missing sequence range. Tracing never fabricates missing events or infers Security decisions. Its health becomes degraded immediately on disconnect or unrecoverable gap, and the warning clears automatically on successful reconnection; the gap diagnostic remains in trace history. Repeated failures mark it degraded immediately.
- A persistence write or rotation error degrades restart durability for the affected stream and is reported separately by `/health`; it never disables the in-memory recorder, discards live trace events, interrupts trace SSE, or affects the bus or Security. The coalesced dirty indication remains pending so persistence can converge after storage recovers. Retry timing and terminal persistence-failure policy remain open.
- On restart restoration, retained trace files must be validated before their contents are admitted into the in-memory history or used to establish sequence continuity. The exact validation, malformed-record handling, reconciliation and recovery policy remain open. Any restoration failure affects restart durability/history restoration only and must not disable live in-memory tracing after the process starts.
- Live trace serving does not tail or read persistence files. File read/validation failures therefore cannot interrupt live trace recording or SSE; they are handled solely as restart-restoration failures under the still-open restoration policy.
- Tracing reconnects with exponential backoff using a 2× multiplier and full jitter. After each unsuccessful attempt, the backoff cap doubles, starting at 1 second and capped at 60 seconds by default; the actual retry delay is selected uniformly at random from zero up to that cap. The initial and maximum delays are deployment-configurable through `trace.reconnectInitialDelay` and `trace.reconnectMaxDelay`. Successful reconnection resets the backoff.
- `authorization.requested` and `authorization.decided` are published on one dedicated internal stream. The core publishes requests and consumes decisions; Security consumes requests and publishes decisions. Both subscribers ignore event types they do not handle, including events they themselves publish. The stream uses the ordinary broker, subscription, publishing, ordering and delivery mechanisms; it is not a separate messaging path.
- Administrator approval and rejection instructions enter Security's existing event-driven authorisation workflow through the dedicated internal authorisation stream. The Admin UI submits the administrator's instruction to the core using the global Admin credential; the core publishes the instruction as `authorization.administratorDecisionRequested` on that stream; Security consumes it, evaluates it and publishes the resulting `authorization.decided` event. The core validates and applies that decision through the already-settled decision mechanism. The core does not directly apply the administrator's authorisation decision or call Security. The instruction payload contains exactly `correlationId`, `subscriberId`, `streamId` and `requestedDecision`. `correlationId` is the original core-generated authorisation request UUID; `subscriberId` and `streamId` identify that exact relationship; `requestedDecision` is exactly `Approved` or `Denied`. The payload does not repeat the original request's operation. Before publishing the event, the core authenticates the bearer credential through the shared credential store and requires the authenticated principal to have the global `Admin` role; the sender's authority is derived from the authenticated credential record, never from request-supplied identity or role fields. The core validates that the payload is a JSON object containing exactly the four required fields and no additional fields; `correlationId` is a valid UUID; `subscriberId` is a valid Application Subscriber ID UUID; `streamId` conforms to the established stream-ID representation; and `requestedDecision` is exactly `Approved` or `Denied` with case-sensitive matching. Invalid or unauthorised instructions are rejected by the core and are not published. The core publishes only the validated four-field instruction. Security independently evaluates it through its existing event-driven policy workflow and publishes the resulting `authorization.decided`; the core validates and applies that decision through the already-settled Security decision mechanism. The core does not directly grant or deny access based on the Admin instruction. Repeated delivery of the same administrator instruction may cause Security to publish repeated `authorization.decided` events; no instruction-specific deduplication registry or second idempotency mechanism is introduced. The core's existing first-valid-decision rule remains authoritative: the first valid decision received for a pending correlation wins, and every subsequent decision for that correlation is discarded and recorded as a duplicate, even if it requests the conflicting outcome. Thus repeated or conflicting instructions cannot reverse or repeat an authorisation effect. If the original request has expired, a resulting decision is discarded and recorded as late using the existing tombstone classification; it cannot revive the request. If the correlation matches neither a pending request nor a retained tombstone, the decision is discarded and recorded as unknown; it cannot create a request or change permissions. Already-acted-on correlations are handled by the existing duplicate-decision rule. No administrator-instruction-specific acknowledgement protocol, retry mechanism, deduplication registry or tombstone policy is added; existing core-owned pending-request and tombstone handling remains authoritative.
- The `authorization.requested` event payload contains `correlationId`, `subscriberId`, `streamId` and `operation`. The core generates `correlationId` as a cryptographically random UUID; `subscriberId` and `streamId` identify the exact relationship under evaluation. `operation` initially supports `subscribe`, `reconnect` and `permissionChange`. The payload contains no bearer credentials, credential hashes or message content.
- All authorisation event JSON field names use camelCase: `correlationId`, `subscriberId` and `streamId`. Security authorisation events follow the broker's existing per-stream sequence ordering. The workflow adds no application-level acknowledgement or retry protocol. If Security is disconnected or its bounded subscription queue overflows before processing a request, the request may go unprocessed; the core never infers approval from delivery failure or silence. The core's configurable 5-second authorisation timeout (by default) applies to ordinary Security authorisation requests that are not awaiting human Admin approval; if Security does not decide within that deadline, the core fails the request closed and removes it from the pending registry. An Application request awaiting Admin approval remains pending for its separate 10-minute approval window, and its original correlation remains eligible for the administrator instruction and resulting Security decision only during that window. If no approval is accepted within 10 minutes, the pending request expires and is no longer eligible for approval; this expiry is not an explicit denial. The ordinary 5-second timeout does not expire that Admin-mediated request. An SSE connection timeout after approval may close the inactive connection but does not change the established authorisation. A decision is accepted at most once for its correlation ID; duplicate decisions are discarded and recorded. The broker's existing queue-overflow behaviour, including subscriber detachment and `stream.gap`, remains in force. Tracing records metadata for every valid `Approved` or `Denied` authorisation outcome, including event type, correlation ID, timestamp, subscriber ID, stream ID and decision; where available from the correlated request, the operation is included. It also records diagnostic metadata for late, duplicate and unknown decisions, including event type, correlation ID, timestamp and discard reason. These records never contain bearer credentials, credential hashes or message payloads. Tracing is observational and is not part of the authorisation decision path.
- The `authorization.decided` event payload contains `correlationId`, `subscriberId`, `streamId` and `decision`. `correlationId` echoes the core-generated UUID from the request; `subscriberId` and `streamId` must exactly match the original request. `decision` is exactly `Approved` or `Denied`; no other values are valid.
- An authorisation request is a logical request carried by an SSE event published by the core through Pulsyflux's existing event infrastructure. Security receives it through its ordinary subscription and publishes its decision as an ordinary event; the core receives and enforces that decision through the same event infrastructure. The SSE event is the transport/delivery mechanism, while the authorisation request is the logical message carried by the event. This is not a direct function call, RPC, HTTP request to Security, or separate messaging channel.
- An ordinary Security authorisation request that is not awaiting human Admin approval and receives no Security decision before its deadline fails closed: the core denies the request and does not grant access by default. The ordinary Security authorisation timeout is configurable and defaults to 5 seconds. When such a request expires, the core immediately removes it from the pending-authorisation registry and retains an expiry tombstone under the bounded retention rules above. A decision matching that tombstone is late and must be ignored; it cannot change subscriber or authorisation state. An Application request awaiting human Admin approval follows a separate 10-minute approval window. Its original correlation remains eligible for the administrator instruction and resulting Security decision only during that window; the ordinary 5-second timeout does not expire it. If no approval is accepted within 10 minutes, the pending request expires and the core closes its pending SSE connection under the existing generic timeout behaviour. Expiry is not an explicit denial, and a later decision cannot authorise or revive the request. If approval was accepted before expiry, a subsequent SSE inactivity timeout may close the connection but does not reverse, revoke or otherwise change the established authorisation. The SSE timeout itself is not an authorisation decision. The first valid decision received for a pending correlation wins: after accepting it, the core removes the request from the pending-authorisation registry and retains a completed-correlation tombstone under the bounded retention rules above. Any subsequent decision matching that tombstone is discarded and recorded as a duplicate; it cannot change subscriber or authorisation state. A Security decision is valid only when its publisher is authenticated as the reserved Security identity, its correlation ID matches a currently pending request, its decision type is explicitly `Approved` or `Denied`, and it arrives before the applicable deadline (5 seconds for ordinary Security authorisation, or 10 minutes for an Application request awaiting Admin approval). Any decision that fails these checks is rejected or discarded and recorded as appropriate. A decision from the authenticated Security identity whose correlation ID matches neither a currently pending request nor a retained expiry or completed-correlation tombstone is discarded and recorded as an unknown decision. It does not create a pending request or change any subscriber's permissions. Each authorisation request receives a unique correlation ID generated by the core as a cryptographically random UUID. Clients and Security cannot choose or supply this ID.
- Security unavailability is handled fail-closed. While Security is unavailable, the core must not grant or authorise new subscriptions or reconnections, and must not process permission-changing decisions or updates without Security's policy decision. Existing authorised connections may continue only under their already-established permissions for the remainder of their existing connection lifecycle; this does not authorise a new connection or reconnection.
- After a Security outage, the Security subscriber recovers by re-authenticating with its existing reserved credential and re-subscribing through the ordinary subscription mechanism. It resumes processing its subscribed events before the core authorises new subscriptions, reconnections or permission changes. The core continues to enforce the fail-closed policy until Security has successfully resumed.
- If the Security subscriber crashes or becomes unavailable, Pulsyflux retries its recovery automatically using the same 2× exponential backoff multiplier and full-jitter algorithm as Tracing. After each unsuccessful attempt, the backoff cap doubles and is capped at Security's independently configured maximum; the actual retry delay is selected uniformly at random from zero up to that cap. The independently deployment-configurable settings are `security.recoveryInitialDelay` (default 1 second) and `security.recoveryMaxDelay` (default 60 seconds). Retries continue without administrator intervention. The core remains fail-closed throughout recovery and does not bypass Security after repeated failures. Normal authorisation resumes only after Security has re-authenticated with its existing reserved credential, re-subscribed through the ordinary subscription mechanism and resumed processing its subscribed events. Successful recovery resets Security's backoff to its initial value.
- Security subscription detachment and unrecoverable `stream.gap` handling are settled: Security becomes unavailable immediately on detachment, including queue overflow, and the core remains fail-closed under the existing policy. Security automatically reconnects through ordinary subscription/SSE replay using its existing reserved credential and Security recovery backoff. It resumes policy processing only after replay proves uninterrupted continuity from its last successfully processed event position; receiving or queueing an event does not advance that position. Successful recovery resets the backoff. On an unrecoverable `stream.gap`, Security records the source stream and missing sequence range, fabricates no events and infers no missed approvals, denials, revocations or permission changes, and does not resume using potentially stale policy state. Security remains unavailable for the rest of that process run, requiring a process restart for recovery because no snapshot/reconciliation protocol exists. This rule applies to stateful policy consumers whose correctness requires complete event history, not observational consumers such as Tracing. More generally, recovery guarantees are determined by subscriber correctness requirements, not internal/external status; no special broker privileges or unlimited queues are introduced.
- Admin is a global role, not scoped to a stream.
- Exactly one Admin credential record exists in memory in every running Pulsyflux instance.
- `PULSYFLUX_ADMIN_TOKEN` is mandatory. If it is missing or empty, Pulsyflux fails startup.
- The Admin token must contain at least 256 bits of cryptographically secure randomness. Pulsyflux hashes it at startup and creates the single Admin credential record; it does not retain the plaintext token.
- The Admin credential cannot be created, approved, duplicated or revoked through subscriber workflows.
- The HTML administration page is an Admin subscriber and authenticates through the same bearer-token hash lookup and role resolution as every other subscriber. There is no separate administrator browser session or authentication bypass.
- Admin can discover and administer every stream. Selecting a stream scopes the operation, not the Admin credential.
- Each Application subscriber installation supplies a stable Subscriber ID UUID and a descriptive Name. Neither is a credential or proof of authorisation.
- Subscribers request Application access by attempting to subscribe; there is no separate subscriber registration workflow.
- New Application access requires administrator approval before message events can flow.
- Approval is scoped to exactly one Subscriber ID + Stream ID relationship and remains valid for the lifetime of the running process.
- Pending Application subscriptions can remain connected through SSE while awaiting approval, for a maximum of 10 minutes. If approval is not accepted within that window, the pending request expires and is no longer eligible for approval; expiry is not an explicit denial. An SSE inactivity timeout after approval may close the connection but does not change the established authorisation.
- An approved pending subscription proceeds on its existing SSE connection.
- Rejection and timeout use the same generic terminal event and do not reveal the administrator's decision.
- Pulsyflux generates 256-bit random bearer credentials for Application, System and Security subscribers using Go's `crypto/rand`, encoded as unpadded Base64 URL-safe text. For Security, the core generates the credential before the subscriber starts, seeds its hash and trusted role record in the shared in-memory credential store, and hands the plaintext credential to the subscriber through a private, one-time in-memory startup handoff.
- Only SHA-256 credential digests are stored in memory; presented tokens are hashed and digest comparisons use a constant-time comparison.
- Credentials are bound to their assigned subscriber identity, role and stream scope.
- Credentials are delivered once, never placed in URLs, and excluded from traces, ordinary logs, errors and the administration interface.
- Returning subscribers present bearer credentials in the HTTP `Authorization: Bearer` header.
- Revocation immediately invalidates the affected Application credential and closes its active SSE connection.
- If an Application subscriber loses its credential during the instance lifetime, an administrator revokes the relationship and the subscriber requests approval again for a new credential.
- All pending requests, credential hashes, role records, sessions and active authentication state are in memory only and disappear on process exit.
- On restart, Pulsyflux recreates exactly one Admin record from the environment variable; Application subscribers must request approval again, and System and Security subscribers are recreated internally.
- Authentication is programming-language independent and uses documented HTTP and SSE interfaces. A Go client library is optional.
- For a valid current-run resume cursor at sequence `seq`, if the oldest retained sequence is `oldestSeq`, retention eviction produces exactly one inclusive `stream.gap` from `seq + 1` through `oldestSeq - 1` only when `seq + 1 < oldestSeq`; retained replay then begins at `oldestSeq`. If `seq + 1 == oldestSeq`, there is no gap. Gaps are never empty or overlapping and never include retained messages. This precise boundary rule is uniform across subscribers, while each subscriber's response to a gap follows its own settled correctness requirements.
- Streams use fan-out pub/sub (every subscriber receives every message) with replay, not competing-consumer queue semantics.
- The stream ID hashes the complete schema document, including annotations such as `title`, `description` and `examples`.
- The service is loopback-only and is not reachable from the LAN.
- An Application subscriber authorised for a Stream ID can read the complete trace for that stream, including retained events from before authorisation and subsequent live events; without `Last-Event-ID`, trace replay starts at the earliest retained event, and supplied cursors follow the existing trace replay and `trace.gap` rules.
- Message content is never included in traces.
- The asynchronous trace-persistence flush target defaults to 50ms and is configurable. An unexpected crash or power loss may lose events since the last successful flush; this restart-durability window is accepted and does not affect live in-memory tracing.
- Persistence scheduling uses one coalesced dirty indication per stream rather than a configurable event-count writer queue. The ordinary Tracing pub/sub subscription remains subject to the broker's standard bounded-queue and gap behaviour.
- Each trace-persistence pass writes a complete retained-history JSON Lines snapshot to a uniquely named temporary file in the same directory as `trace.jsonl`, flushes and syncs it, closes it, and atomically replaces the active file without deleting the active file first. The previous committed snapshot remains available if writing or replacement fails; incomplete temporary files are not committed snapshots. Where backups are configured, the previous committed snapshot is preserved before replacement and older backups are pruned only after the new snapshot commits. Directory metadata is synced where supported, with platform-specific atomic-replacement semantics accounted for. Atomic replacement is the commit point; this protects snapshot integrity but does not change the accepted crash-loss window. No manifest or second persistence index is introduced.
- There is no public stream-clear or stream-delete operation in the first release.
- SSE replay uses the standard `Last-Event-ID` HTTP header only. No `?lastEventId=` query parameter or alternative query-based replay mechanism is supported.
- The stream-monitor activity sparkline is calculated in the browser from the change in each stream's accepted publish count between successive `GET /v1/streams` polls. Rejected publish counts remain separate and are not included in the sparkline.
- When the stream monitor filters by subscription, it shows events belonging to that subscription and retains corresponding `publish.accepted` context events when they share the same `runId` and message `seq` with a visible `delivery`. It hides unrelated publishes and deliveries from other subscriptions; filtering changes presentation only and preserves the trace records and `traceSeq` order.
- On the stream monitor, Pause closes the selected stream's trace SSE connection and retains the most recently received SSE event ID as the resume cursor. For trace events this is their `traceSeq`; for a `trace.gap` control event it is the event's `toTraceSeq` ID. Resume reconnects to the selected stream using that cursor in the standard `Last-Event-ID` HTTP header. The server replays retained in-memory events after the cursor and reports any events evicted by in-memory retention through the existing `trace.gap` mechanism; no client-side event backlog or new replay mechanism is introduced.
- For a syntactically valid trace resume cursor older than the earliest retained `traceSeq`, the server emits exactly one `trace.gap` with inclusive bounds `cursor + 1` through `earliestRetained - 1`, then replays from `earliestRetained`. The gap event's SSE ID is `toTraceSeq`, preventing the same missing range from being reported again after reconnect. This does not change malformed-cursor fallback (replay all retained events without a gap) or future-cursor behaviour (wait for future events without a gap).