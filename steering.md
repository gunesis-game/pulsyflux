
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

#### Snapshot scheduling

This is a settled Core scheduling model:

- Core uses one global periodic snapshot scheduler for all streams whose persistence policy is enabled.
- The scheduler checks eligible streams and snapshots only streams whose persisted state has changed since their last successfully completed snapshot.
- There are no separate per-stream snapshot timers, and publication does not trigger a synchronous snapshot.
- The snapshot interval is part of the policy provided by each stream class. Each persistence-enabled stream uses its stream class's configured interval unless that stream has an explicit per-stream override.
- Per-stream overrides change only the effective due interval; they do not create per-stream timers. The one global scheduler evaluates each eligible stream against its effective interval.
- Snapshot intervals are fixed policy values, not adaptive values. Core does not automatically increase or decrease a stream's interval in response to publication activity or observed load. A per-stream override is likewise a fixed configured value unless explicitly changed by configuration.
- Application streams use a fixed 5-second snapshot interval. System streams use a fixed 1-second snapshot interval. Admin streams use a fixed 10-second snapshot interval.
- This scheduler operates on Broker-owned stream state. It does not change Broker's ownership of authoritative history or the settled rule that publication acceptance does not wait for snapshot completion.

#### Bounded crash-loss target

The user approved a **15-second crash-loss target** for persistence-enabled streams:

- Under healthy persistence operation, the system targets no more than 15 seconds of accepted messages being newer than the latest successfully completed durable snapshot at the point of abrupt process failure.
- Measure the window as elapsed wall-clock time from completion of the latest successful snapshot for that stream to the failure. The target therefore requires a changed persisted stream's next snapshot to complete within 15 seconds of the preceding successful completion.
- The 15-second value is an operational target, not an unconditional durability guarantee. Snapshot write, validation or promotion failures, an unavailable or slow persistence backend, or process failure can cause the target to be missed. Such conditions must be surfaced as a durability/degraded-state failure; they must not be represented as meeting the target.
- The target does not change the approved fixed class intervals (Application 5 seconds, System 1 second, Admin 10 seconds), optional fixed per-stream overrides, one global scheduler, asynchronous publication acceptance, or atomic snapshot promotion. The scheduler and persistence execution must allow for snapshot completion time; an interval becoming due is not itself a completed snapshot.

### Broker

Broker is the one shared messaging and stream infrastructure. It owns registered streams and their authoritative message queues/logs, accepts publications, manages subscriptions, performs fan-out routing and dispatch, enforces delivery/backpressure rules, and owns message ordering, retention, replay and stream lifecycle. Broker also owns the message data from which stream snapshots and durable persistence are produced. Snapshotting and persistence must not introduce a second authoritative message store.

A stream is the communication boundary. Components publish messages to streams and subscribe to streams. A publisher does not need to know which components subscribe; a subscriber does not need to know which component published. One stream may have many publishers and many subscribers. Fan-out semantics mean each authorised subscriber receives its own delivery of each message; this is not competing-consumer work-queue semantics.

#### Stream registration and lifecycle

This is a settled Broker lifecycle contract:

- A stream is identified by the hash of its canonical JSON Schema. Registering a schema whose identity is already registered is idempotent and returns the existing `streamId`; it does not create a duplicate stream.
- A registered stream's schema is immutable; Broker does not modify a schema in place.
- A changed schema produces a different canonical schema hash and therefore a different `streamId` and stream. The existing stream remains registered with its authoritative history and subscriptions.
- Moving between schema versions is an explicit application-level migration. Broker does not implicitly migrate messages, history, subscriptions or stream identity between schema versions.
- Re-registering an existing schema does not clear or replace its authoritative message history, reset its sequence, alter its retention policy, or interrupt existing subscriptions.
- Streams are not deleted through an ordinary runtime operation in the initial version. A registered stream remains registered for the lifetime of the Broker state.
- Streams do not have a reset operation. Clearing history or restarting a stream's sequence is not a supported lifecycle action.
- Any future need to retire, migrate or replace a stream must be designed as an explicit migration/lifecycle capability; it must not be approximated by deleting and re-registering the same schema identity.
- The `schema.registered` trace event is emitted only when a new schema is admitted, not for an idempotent re-registration.

This keeps schema identity, authoritative history and sequence continuity stable across participants and repeated registration attempts.

#### Delivery confirmation and resume-cursor ownership

This is a settled Broker delivery contract:

- Each delivered message exposes its authoritative Broker stream sequence as its SSE event ID.
- A subscriber's client-supplied `Last-Event-ID` is the authoritative resume cursor. On reconnection, the subscriber supplies the last sequence it received, and Broker resumes from that cursor using the stream's authoritative retained history.
- Successful server-side writing and flushing of an SSE frame is a transport-delivery observation only. It does not establish that the client received, acknowledged or processed the message.
- Broker does not add a separate per-message acknowledgement protocol or a subscriber-owned message history for this purpose.
- On reconnect, Broker replays the events still present in the stream's authoritative retained history after the subscriber's supplied cursor. If older events have been evicted, they are unavailable; Broker does not emit a separate retention-gap control event. Replay continues from the earliest retained event.
- This contract applies uniformly to Application, Security, Tracing, Admin and other subscribers. It does not weaken domain-specific correctness requirements; Security's established strict continuity and fail-closed behaviour remain in force.

#### Subscription identity and duplicate subscriptions

This is a settled Broker subscription contract:

- Every successful subscription registration creates a new, unique Broker-generated `subscriptionId`. Subscription IDs are distinct from subscriber identities and stream IDs.
- A subscriber may hold multiple simultaneous independent subscriptions to the same stream. Repeated registration does not silently reuse or deduplicate an existing subscription.
- Each subscription has its own lifecycle and bounded temporary delivery buffer. Overflow or closure of one subscription affects only that subscription, not other subscriptions belonging to the same subscriber.
- Each subscription receives its own fan-out delivery of every message for which it is authorised, preserving the stream's authoritative sequence. Multiple subscriptions do not change the stream's ordering or create competing-consumer semantics.
- Reconnection is a new delivery attachment and resumes from the client-supplied `Last-Event-ID` against Broker's authoritative retained history. It does not create a second message history. The subscription identity/lifecycle relationship across reconnects must be represented consistently in the public contract; the reconnect does not imply reuse of an old live subscription.
- Security authorisation remains based on subscriber identity, role and stream scope; multiple subscriptions grant no additional permissions. Revoking a subscriber–stream relationship terminates every active subscription covered by that relationship.
- Tracing correlates subscription lifecycle and delivery observations using the unique `subscriptionId`.

#### Subscriber delivery-buffer overflow

This is a settled Broker delivery policy:

- Each subscriber has its own bounded, temporary delivery buffer. If that buffer becomes full, Broker detaches only the affected subscription.
- A full subscriber buffer must not block publication to the stream or delivery to any other subscriber.
- Broker must not silently discard messages to make room in a full delivery buffer.
- Detaching a subscription does not alter, truncate or otherwise change the stream's authoritative message history, retention or sequence.
- The subscriber may reconnect and resume from its last successfully delivered stream sequence. Replay uses Broker's authoritative retained history, not the former delivery buffer.
- If retention has overtaken the subscriber's resume cursor, Broker replays from the earliest event still retained. Events already evicted are unavailable; no separate gap notification is sent.
- This policy applies uniformly to Application, Security, Tracing, Admin and other subscribers. Domain-specific correctness and failure handling remain the responsibility of the relevant participant; in particular, Security's established strict continuity and fail-closed requirements remain in force.

#### Message ordering and sequence ownership

This is a settled Broker contract:

- Each Broker stream has its own monotonically increasing message sequence.
- The first accepted message in a stream receives sequence number `1`.
- Broker assigns the sequence atomically at the point the message is accepted and appended to that stream's authoritative queue/log. Concurrent publishers therefore cannot create duplicate or ambiguous sequence numbers.
- A rejected publication does not consume a sequence number.
- Every subscriber receives messages in the stream's sequence order. Fan-out deliveries of the same published message retain the same stream sequence; sequence identity is independent of subscriber count.
- Broker establishes the order once. Fan-out, delivery tracking, replay, tracing all refer to that same stream ordering; they must not introduce competing sequence authorities.
- Sequence numbers are scoped to a Broker process run. The run identifier (`runId`) distinguishes message identities across process restarts.
- Persistence and restoration preserve unambiguous message identity and ordering. When a persisted stream is restored, its sequence continues from the last restored message: the next accepted message receives the previous sequence plus one. The new process uses a new `runId`, distinguishing message identities across process runs while the stream sequence remains continuous. If no messages are restored for a stream, its first accepted message receives sequence number `1`.


#### Retention ownership and authoritative stream history

This is a settled Broker ownership decision:

- Broker owns each stream's authoritative message history: the official ordered record of messages accepted into that stream. It is the single source of truth for the stream's retained messages.
- Broker applies the configured retention policy to that authoritative history. This decision establishes ownership, not a universal retention duration or a specific eviction threshold; those details remain subject to the stream-class policy and any per-stream override.
- Replay reads from the authoritative stream history, subject to the messages still retained under the applicable policy. Subscriber delivery buffers are not replay sources and do not determine authoritative-history retention.
- Per-subscriber delivery buffers remain bounded, temporary delivery state. They may contain only a subset of the stream's messages and may be drained, discarded or lost independently without changing the authoritative history.
- There is one authoritative history per stream, shared as the source for all authorised subscribers; there are no subscriber-specific retained histories or subsystem-owned copies.


#### Publication acceptance and crash durability

This is a settled durability contract:

- A successful publication means Broker has accepted the message and appended it to the stream's in-memory authoritative history. It does not mean the message has been durably persisted.
- For streams whose persistence policy is enabled, Core may persist snapshots asynchronously; publication acceptance does not wait for a durable snapshot.
- After an abrupt process failure, accepted messages newer than the latest successfully completed durable snapshot may be lost.
- On restart, Broker restores the latest successfully persisted snapshot available for each opted-in stream before making that stream available for normal publication, subscription or replay. Its next sequence continues from the last restored message under the existing sequence/runId rules.
- Streams without persistence opt-in remain non-persistent. Security authentication and authorisation state remains non-persistent under the existing policy.
- This contract distinguishes in-memory acceptance from durable persistence. The snapshot boundary and atomic-promotion models are settled separately: each snapshot captures a consistent point-in-time state of one stream, identified by its highest included sequence; messages accepted after that boundary are excluded and remain eligible for a later snapshot. A new complete snapshot is persisted separately and promoted atomically, preserving the previous completed snapshot if writing or promotion fails. The approved 15-second crash-loss target is specified under Snapshot scheduling; the scheduling model is settled separately below.

#### Relationship between authoritative history, snapshots and persistence

This is a settled architectural relationship:

- Broker remains the owner of each stream's authoritative message history, including accepted messages and their ordering.
- Core provides the shared snapshot and persistence mechanisms and policies. These mechanisms operate on Broker-owned stream and message state; they do not own a separate message history or create a second source of truth.
- Each snapshot captures a consistent point-in-time state of one Broker-owned stream, identified by the highest stream sequence included in that snapshot. Messages accepted before or at that boundary are included; messages accepted after it are excluded even if the snapshot is still being written, and remain in Broker's authoritative history for a later snapshot. A snapshot is a representation of Broker state, not another independently managed queue or message store.
- A snapshot is successfully completed only after its complete captured state has been persisted. A partially written or incomplete snapshot is not treated as the latest successfully completed durable snapshot.
- Persistence durably stores only the snapshot data selected by the applicable stream-class persistence policy and any explicit per-stream override.
- On restoration, persisted stream data is restored into Broker's authoritative stream state before that state is made available for normal publication, subscription or replay.
- Subscriber delivery buffers are temporary delivery state. They are not authoritative history and are not the source from which stream snapshots or persistence are produced.
- This decision establishes ownership and the relationship between the mechanisms. Publication acceptance versus durable persistence, including the possibility of losing messages newer than the latest completed durable snapshot after abrupt failure, is settled separately above. The global snapshot scheduling model, snapshot boundary model and atomic-promotion model are settled separately below; the approved 15-second crash-loss target applies as specified under Snapshot scheduling.


Internal and external messaging use the same Broker stream model and core messaging semantics. External access is an exposure and authorisation boundary, not a separate messaging architecture. Transport adapters may differ, but they must not create parallel stream, queue, subscription or persistence models.

### Messaging participants

Application, Security, Tracing and Admin are independent participants in the same infrastructure. Each may publish and subscribe, subject to its role, identity, scope and authorised stream relationships. Their responsibilities differ; their messaging infrastructure does not.

- **Application:** the client-facing/external application participant. External clients use the documented Application-facing API and its approval and stream-scope rules. Application may publish to and subscribe to streams required by its application responsibilities.
- **Security:** an internally provisioned participant. It subscribes to streams carrying identity, access and subscription-authorisation requests and publishes its security and authorisation outcomes. Security owns the policy decision and its validation; it does not orchestrate Admin's registration workflow. Broker transports the messages but does not make Security decisions.
- **Admin:** an internally authenticated administrative participant. It owns and orchestrates administrative subscription registration. It waits for the required positive Security authorisation outcome, then initiates the authorised registration with Broker. Admin does not make Security policy decisions, and Broker does not originate, approve or hold pending administrative registration requests.
- **Tracing:** an internal participant that subscribes to relevant source streams and publishes tracing/observability events to appropriate stream(s). Its external trace access or integration surface consumes the same Broker-owned stream data under the documented authentication and authorisation rules. Tracing does not own a parallel message history, queue or persistence engine.
- **Admin:** an internally authenticated administrative participant. It publishes administrative instructions/events and subscribes to streams needed for administration and observation. It coordinates subscription registration after the required positive Security authorisation outcome. Its global administrative authority comes from its trusted identity and role, not from a separate messaging path.

These are role and responsibility distinctions, not separate broker implementations. A participant may both publish and subscribe. Exact stream permissions remain governed by authenticated identity, role and scope.

### Communication rule: no direct component-to-component calls

Application, Security, Tracing, Admin and other domain components must not communicate by directly calling one another's methods, RPC endpoints, private channels or bespoke integration interfaces when exchanging domain messages or requesting domain action. They communicate by publishing to and subscribing from Broker streams. The receiving component interprets and processes the message according to its own responsibility and may publish a resulting message to another stream.

A publisher must not depend on knowing its subscribers. A subscriber must not depend on knowing the publisher's implementation. Broker owns delivery; domain components own message meaning and processing. Ordinary constructor dependency injection and calls to shared Core infrastructure are not domain messaging and remain valid.

Do not add a generic component-to-component messaging abstraction. Request/decision workflows, including Security authorisation, are logical messages carried on streams and use ordinary Broker publishing and subscription. HTTP/SSE is the documented external transport/API surface; it does not create a private direct-call channel between internal components.

### Streams, queues, snapshots and persistence

A stream is a schema-identified, ordered message history managed by Broker. Its queue/log is the authoritative in-memory collection of published messages and the source for fan-out, replay and retention. Per-subscriber delivery queues are bounded delivery buffers; they are not independent stream histories and do not own the published messages.

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

Core provides the shared in-memory snapshot and persistence model used for Broker-owned stream state. Every stream has a complete in-memory snapshot governed by its JSON Schema, regardless of whether any part is persisted. The snapshot represents the corresponding Broker-owned stream/queue state; it is not a second message store. The persistence policy selects which schema-defined properties are written to durable storage. A stream's role determines its stream class and class-level persistence policy; individual streams may override that policy. A class with no explicit persistence policy is non-persistent by default. Snapshot availability in memory is never conditional on persistence. The stream-class policy also provides the snapshot interval for streams in that class; an individual stream may explicitly override that interval. Core still uses one global scheduler, which evaluates each persistence-enabled stream against its effective interval rather than creating per-stream timers.

Broker remains the owner of stream identity, queues, messages, ordering, retention, subscriptions and replay. Core owns the reusable snapshot/persistence mechanism and policy. Snapshot and persistence work from Broker-owned messages and stream state; restoring persisted data restores that state to Broker before it is made available for normal publication, subscription or replay. No subsystem may create an independent queue, message history, file store or duplicate state store for data already represented by Broker streams and Core snapshots.

Application, System, Security and Admin stream state follows the shared Core snapshot/persistence policy. This does not make all control-plane or authentication state durable. Security state, including subscriber identities, roles, credential hashes, pending approvals and authorisation state, is represented in the shared in-memory snapshot model according to its schema. Security snapshots are not persisted; sensitive and transient Security properties are excluded from durable storage by policy. Plaintext bearer credentials are not retained in snapshots; credential hashes are held where required for verification. This does not change the existing authentication/authorisation lifecycle or bootstrap rules.

A successful publication means in-memory acceptance by Broker, not durable persistence. For opted-in streams, Core may persist snapshots asynchronously, without making publication wait for a durable snapshot. After an abrupt process failure, messages accepted after the latest successfully completed durable snapshot may be lost. On restart, Broker restores the latest successfully persisted snapshot available for each opted-in stream before normal use; sequence continues from the last restored message under the existing sequence/runId rules. Streams without persistence opt-in and Security authentication/authorisation state remain non-persistent. In-memory snapshots are lost when the process exits. Only schema properties selected by an explicit persistence policy may be restored after restart. Persistence and snapshots must not create a second authoritative message history beside Broker's queue/log.

### Docs UI

Served at `/`, with a "Try it" console generated from the contract.

### Tracing

### Governing architecture — Tracing is a Broker participant

Tracing is an internally provisioned System-role messaging participant. It subscribes to relevant source streams and publishes trace/observability events to the configured tracing stream or streams. It may also expose trace data through the documented external trace API and monitor UI. Internal observation and external trace access use the same Broker-owned stream data and the existing authentication/authorisation model; neither creates a separate trace store or messaging path.

Tracing is both a publisher and a subscriber. It uses the ordinary Broker publishing and subscription mechanisms, ordinary identity and credential verification, and the standard Security authorisation workflow. Its System identity does not grant a bypass, unlimited queue, special delivery semantics or direct access to another component's implementation. Application, Security, Admin and future participants interact with Tracing only through streams and documented external interfaces.

Broker owns tracing-stream registration, message queues/logs, message ordering, fan-out, bounded delivery queues, retention, replay, gap reporting and message lifecycle. Core's shared snapshot/persistence mechanism applies to tracing streams according to the stream class and explicit persistence policy. Tracing does not own a parallel per-stream history, queue, recorder store, snapshot model, persistence worker, trace file rotation system or restore path. The trace API reads Broker-owned tracing-stream messages; it does not tail files or read a separate trace database.

### Trace events

Trace events are ordinary messages published to a tracing stream and validated by that stream's JSON Schema. The TraceEvent schema remains part of api/asyncapi.yaml and its Go type is generated like every other wire type. Trace events use the ordinary Broker message envelope and are subject to the same publication validation, stream ordering, retention, replay, persistence policy and subscriber delivery rules as other messages.

Trace records describe what happened to publishes and subscriptions, in the order observed. The existing TraceEvent domain fields are retained: runId identifies the process run; at is a UTC RFC 3339 timestamp with nanoseconds and is informational; kind identifies the event kind; streamId identifies the relevant source stream; subscriptionId identifies the relevant subscription where applicable; seq identifies the source message sequence where applicable; and detail contains kind-specific metadata. The existing traceSeq field and its relationship to Broker's stream seq are not yet reconciled. Broker's stream sequence remains authoritative for message ordering, retention and replay; traceSeq must not become a second stream-ordering or replay mechanism. The exact field compatibility and migration decision must be made in api/asyncapi.yaml before implementation. Message payload data and bearer credentials or credential hashes are never copied into trace records.

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

The earlier trace-specific TraceEvent.traceSeq, traceSeq-based Last-Event-ID cursor, trace-specific gap event and independent trace retention rules require reconciliation with Broker's authoritative stream seq, standard Last-Event-ID cursor, retention-gap notification (removed) event and retention rules. Preserve any trace-specific information that is genuinely required by the public trace contract, but it must map consistently to Broker message identity and must not create a second ordering, retention or replay source of truth. This mapping is an explicit pre-implementation contract decision; do not silently invent it.

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

- Filters by event kind and subscription. When a subscription filter is active, the page shows events belonging to that subscription and retains the corresponding `publish.accepted` events as context when they share the same `runId` and message `seq` with a visible `delivery`. It hides unrelated publishes and deliveries from other subscriptions. Filtering changes presentation only; it does not change Broker messages or their stream order.
- Pause closes the selected stream's trace SSE connection and retains the most recently received Broker SSE event ID as the resume cursor.
- Resume opens a new trace SSE connection using the retained cursor in the standard `Last-Event-ID` HTTP header.
- Replay follows Broker's standard retained queue and cursor semantics.
- Retention gaps follow Broker's ordinary replay semantics. Trace-specific gap notifications are not implied by Broker replay; any such domain-level event would require a separate explicit contract decision.
- Jump to the selected Broker stream sequence.

The page consumes trace messages through the documented Broker-backed trace API only. It does not read persistence files, maintain a second history or invent a private replay mechanism.

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

An administrative approval/registration action may proceed only after Security has issued the required positive authorisation outcome for the Subscriber ID + Stream ID scope. Admin then initiates the authorised registration with Broker. Broker creates the runtime subscription and its unique `subscriptionId`; it does not make the policy decision, originate the registration workflow or hold an unapproved pending registration. The existing pending-request deadline, credential-delivery and connection rules remain governed by the settled Security authorisation contract.

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

### Settled Core snapshot scheduling model

The user approved one global periodic scheduler in Core for all streams opted into persistence:

- The scheduler checks eligible streams and snapshots only streams whose persisted state has changed since their last successfully completed snapshot.
- There are no per-stream timers and no synchronous snapshot on each publication.
- The snapshot interval is part of each stream class's policy, with an optional per-stream override. The global scheduler evaluates each persistence-enabled stream against its effective interval; this does not create per-stream timers.
- Intervals are fixed configuration values, not automatically adjusted in response to publication activity or observed load. Explicit configuration changes may change a value; automatic adaptive scheduling is not part of the design.
- Core performs this shared scheduling over Broker-owned state. Broker remains authoritative, and the previously settled asynchronous publication/durability contract remains unchanged.

The 15-second crash-loss target is settled under Snapshot scheduling.

### Settled Broker snapshot boundary model

The user approved a consistent per-stream point-in-time snapshot boundary:

- Each snapshot captures one stream's state at a defined point, identified by the highest Broker stream sequence included.
- Messages accepted before or at that boundary are included. Messages accepted after it are excluded from that snapshot, even if persistence is still in progress; those messages remain in Broker's authoritative history and are eligible for a later snapshot.
- A snapshot is considered successfully completed only after its complete captured state has been persisted.
- The boundary is per stream and does not make publication wait for snapshot writing. It works with the settled global periodic Core scheduler, which snapshots changed persistence-enabled streams.
- The atomic-promotion model is settled separately: write and validate a complete new snapshot separately, then atomically promote it while retaining the previous completed snapshot until promotion succeeds. The approved 15-second crash-loss target is settled under Snapshot scheduling.

### Settled Core snapshot write atomicity

The user approved an atomic-promotion model for snapshots persisted by Core:

- Write each new snapshot separately from the currently completed snapshot, using a temporary file or equivalent separate storage object.
- Validate that the complete snapshot has been written successfully before promotion.
- Promote the new snapshot atomically as the latest completed snapshot.
- Keep the previous completed snapshot available until the new snapshot's promotion succeeds.
- If writing, validation or promotion fails, the previous completed snapshot remains the recovery point; an incomplete snapshot must not replace it.
- This is the shared Core persistence behaviour for persistence-enabled Broker streams. It does not change Broker's authoritative in-memory history, per-stream snapshot boundary, global scheduling model or asynchronous publication-acceptance contract.
- This decision establishes atomic promotion and recovery-point preservation, not a bounded crash-loss window or a particular storage technology.

### Settled Broker publication acceptance and crash durability

The user approved the distinction between successful in-memory publication and durable persistence:

- A successful publication means Broker accepted and appended the message to its in-memory authoritative stream history; it is not a durability guarantee.
- For streams opted into persistence, Core may persist snapshots asynchronously. Publication does not wait for durable snapshot completion.
- After abrupt process failure, messages accepted after the latest successfully completed durable snapshot may be lost.
- On restart, Broker restores the latest successfully persisted snapshot available for each opted-in stream before normal use. Sequence continues from the last restored message under the established sequence and `runId` rules.
- Streams without persistence opt-in and Security authentication/authorisation state remain non-persistent.
- Snapshot interval values and the 15-second crash-loss target are settled. Stream-class interval policy with optional per-stream overrides, the global scheduling, per-stream snapshot boundary and atomic-promotion models are settled separately.

### Open contract decision — Broker control-event schemas and stream-schema isolation

The current discussion is about the wire contract for Broker-generated retention-gap notification (removed) notifications. The user raised an important constraint: every stream is identified by and governed by its own strict JSON Schema, so an external client must not receive control data that it mistakenly treats as an ordinary stream message and validates against that stream's schema.

Settled context:
- Each stream has its own strict JSON Schema, identified by the canonical schema hash.
- Ordinary published stream-message data must conform to that stream's schema.
- retention-gap notification (removed) is a Broker-generated SSE control event, not a published stream message. It is not appended to stream history or persisted, has no Broker stream sequence and carries no SSE id.
- Its agreed payload fields are streamId, requestedAfterSeq and earliestAvailableSeq.
- Broker remains completely unaware of stream classes, domain workflows and event meaning.

Open decision — the user has not yet approved whether Broker-generated control events should have their own explicitly defined schemas, separate from every stream's JSON Schema. The assistant recommended yes, so external clients can validate/handle Broker control events against a distinct Broker-owned contract and never confuse them with stream messages. The user has not agreed or disagreed yet. Do not record this proposal as settled.

The exact SSE wire representation is also not settled. The assistant suggested a named SSE event (event: retention-gap notification (removed)) with JSON in data:, and no id:; this remains a proposal only. Resolve the control-schema separation question first, one decision at a time, before finalising wire encoding or client validation behaviour.

### Settled Broker delivery confirmation and resume-cursor ownership

The user approved using the client's `Last-Event-ID` as the authoritative resume cursor without introducing explicit per-message acknowledgements:

- Each delivered message carries its authoritative Broker stream sequence as its SSE event ID.
- On reconnect, the subscriber supplies the last sequence it received through `Last-Event-ID`; Broker resumes from that cursor using authoritative retained stream history.
- A successful server-side write and flush is only a transport-delivery observation. It is not proof that the client received, acknowledged or processed the message.
- No separate acknowledgement protocol or subscriber-owned message history is introduced.
- Existing Broker replay and explicit ordinary replay behaviour govern cursor recovery and retention gaps.
- The rule applies to all participants and preserves domain-specific correctness requirements, including Security's strict continuity and fail-closed behaviour.

### Settled Broker retention model and eviction ordering

The user has approved a three-dimensional retention policy for each Broker-owned authoritative stream history:

- Retention has three independent, cumulative hard limits: maximum age, maximum retained message count and maximum retained storage size.
- Retention limits are configured by stream class, with optional per-stream overrides.
- Maximum age is measured from Broker acceptance time, not replay or delivery time. Messages older than the configured maximum age are eligible for eviction.
- If the retained message count or retained storage size exceeds its configured limit, Broker evicts the oldest retained messages first.
- When one or more limits are exceeded, Broker continues oldest-first eviction until all configured limits are satisfied. Meeting one limit does not exempt retained messages from either of the other limits.
- These limits apply to Broker's authoritative stream history, not to per-subscriber delivery buffers.
- The illustrative values 7 days, 100,000 messages and 500 MB are not approved defaults.

The model and oldest-first ordering are settled. The enforcement model is also settled as hybrid:

- On every accepted publication, Broker enforces retention against that stream's authoritative history. This pass removes messages already past the age limit and enforces the configured message-count and storage-size limits, evicting oldest-first until all configured limits are satisfied.\n- A periodic maintenance pass independently checks streams for age expiry, including idle streams that receive no new publications. It evicts expired messages even when no publication occurs.\n- Both paths operate on the same authoritative history and apply the same retention rules; neither changes subscriber delivery-buffer ownership or semantics.\n- A single global periodic maintenance pass runs once per minute and checks streams for age expiry, including idle streams. This is a system-wide pass, not a separately scheduled timer for each stream.\n\nStorage-size accounting is settled: Broker measures each message using the byte length of its canonical serialized representation, including the complete Broker message envelope and stored message metadata. The same per-message byte measurement is used both to calculate a stream's total retained storage size and to enforce the individual-message maximum. The accounting measures serialized authoritative message data, not subscriber delivery buffers, indexes, allocator overhead or other in-memory implementation costs. A separate maximum individual-message size is settled: it is configured by stream class with optional per-stream overrides; Broker checks it before appending a publication to authoritative history; an oversized publication is rejected without evicting existing retained history and without consuming a stream sequence number. Replay/gap behaviour at eviction boundaries is also settled at the semantic level: when a replay or resume cursor predates the earliest message still retained, Broker must explicitly report a retention gap using the ordinary replay semantics, identify the earliest available sequence, and resume replay from the earliest retained message. It must not silently return an incomplete history. With no resume cursor, replay begins at the earliest retained message without implying a client-specific gap. The agreed payload fields are `streamId`, `requestedAfterSeq` and `earliestAvailableSeq`; exact wire representation remains a public-contract detail.


### Settled Admin/Security subscription-registration responsibility

The user approved the responsibility boundary for subscription registration:

- Security owns the authorisation decision for the requesting subscriber, role and stream scope. It evaluates the request and publishes the decision through the established Broker-stream workflow. A positive Security outcome is a prerequisite; Broker does not make or infer this policy decision.
- Admin owns and orchestrates the administrative registration workflow. After receiving the required positive Security outcome, Admin initiates the authorised subscription registration with Broker.
- Broker does not originate registration requests, decide whether a request is authorised, or create/hold a pending administrative registration while Security is deciding. Admin invokes Broker's generic subscription-management capability; Broker performs only generic subscription mechanics, assigns its unique Broker-generated `subscriptionId`, and manages delivery, buffering, replay and lifecycle under the generic Broker contract. Broker does not inspect or validate the domain authorisation, interpret the Admin instruction, or understand the workflow or stream class.
- This separates workflow ownership (Admin), policy authority (Security) and runtime messaging mechanics (Broker). The request, decision and registration coordination remain messages on Broker streams; no direct component-to-component call or RPC is introduced.
- Existing settled Security request/decision schemas, validation, deadlines, timeout handling, tombstones, recovery, credential handling and fail-closed behaviour remain in force. This responsibility clarification does not silently redesign those contracts.

### Settled Admin registration/administrative-action stream class

The user approved that Admin's registration and administrative-action messages use the **Admin stream class**:

- Admin publishes the administrative registration/action message on the Admin-class stream because it represents an administrative action owned and initiated by Admin.
- Admin initiates the authorised registration only after receiving the required positive Security outcome for the Subscriber ID + Stream ID scope.
- Security remains the policy authority and publishes its decision on the Security-class stream. Admin remains the owner of registration workflow and orchestration.
- Admin invokes Broker's generic subscription-management capability to create the runtime subscription. Broker performs only generic subscription mechanics, generates the unique `subscriptionId`, and manages delivery, buffering, replay and lifecycle. Broker does not validate authorisation, inspect the Admin instruction, interpret stream classes or workflow, make policy decisions, originate the registration workflow or hold pending registrations.
- This classification does not change established request/decision or registration contracts, validation, deadlines, timeout handling, tombstones, recovery, credential handling or fail-closed behaviour. No direct component call or separate messaging channel is introduced.

### Settled Security authorisation-decision stream class

The user approved that Security's authorisation decisions use the **Security stream class**:

- Security publishes its authorisation outcomes on the Security-class decision stream because the outcome is the expression of Security's policy evaluation and belongs to Security's domain responsibility.
- Admin subscribes to the relevant Security-class decision stream and, only after the required positive outcome for the Subscriber ID + Stream ID scope, initiates the authorised registration with Broker.
- The decision remains governed by Security's existing request/decision schemas, validation, deadlines, timeout handling, tombstones, recovery, credential handling and fail-closed behaviour. Security authentication and authorisation state remains non-persistent under the existing policy.
- This classification does not transfer policy ownership to Admin or registration ownership to Security. Broker transports the messages and does not make policy decisions or orchestrate registration. No direct component call or separate messaging channel is introduced.

### Settled incoming subscription/access request stream class

The user approved that incoming client-facing subscription/access requests use the **Application stream class**:

- The request originates from the client-facing Application participant and represents an Application access request, so its stream class is Application.
- Security subscribes to the relevant Application-class request stream and evaluates the request under its existing identity, role, scope, validation and authorisation rules. Security's subscription to that stream does not change the message's class or transfer policy ownership to Application.
- This classifies the incoming request by its origin and domain purpose, not by the identity of its consumer. It does not change the established request schema, Security decision contract, deadlines, timeout handling, tombstones, recovery, credential handling or fail-closed behaviour.
- The request remains a message on the shared Broker stream infrastructure. No direct Application-to-Security call or separate request channel is introduced.

### Settled clarification — Broker is workflow- and stream-class-agnostic

The user explicitly clarified that Broker must be completely unaware of domain stream classes and participant workflows. Broker is shared infrastructure used by **all** participants and **all** streams, not a final stage at the end of an Application/Security/Admin workflow.

- Application, Security, Admin and Tracing independently publish and subscribe through the same Broker infrastructure. Broker is involved in every stream exchange, not only in the final subscription-registration step.
- Messages/events flow from publishers through Broker to interested subscribers. Each domain participant interprets and handles the message types relevant to its own responsibilities.
- Broker provides generic messaging capabilities: accepting publications, maintaining authoritative stream history and ordering, routing/fan-out, subscription mechanics, delivery buffers, retention and replay. It does not interpret message meaning or domain workflow.
- Broker does not know or reason about Application, Security or Admin stream classes; it does not inspect commands for domain meaning, validate authorisation, check publisher roles, decide whether registration is permitted, or orchestrate participant workflows.
- Security owns the authorisation decision. Admin owns the workflow and initiates subscription creation after a positive Security outcome. Admin uses Broker's generic subscription-management capability; Broker performs the generic mechanics only.
- The diagrammatic model is participants above/around a shared Broker infrastructure, with every participant's publish and subscribe paths passing through that same infrastructure. Broker is not a downstream workflow participant.
- **Settled event-only messaging model:** Pulsyflux uses events as its universal messaging concept. Every publication publishes an event carrying message data; every subscription listens for events and consumes their data. An event may express a request for an action, a decision, a notification or a fact. These are meanings carried by event data and interpreted by the receiving participant, not separate command/event message categories. There is no separate command message kind or command bus. Broker transports and delivers events using generic stream mechanics and remains unaware of their meaning or the participants' workflows.

### Settled foundational architecture — Broker streams are the only component communication model

The user has clarified and settled the macro-architecture: Application, Security, Tracing and Admin are independent participants in one shared Broker infrastructure. Each can publish to and subscribe to streams according to its responsibility, authenticated identity, role and scope. They are not merely subscribers; each may be both publisher and subscriber. Components communicate with one another through Broker streams, not direct method calls, RPC, private channels or a separate component-to-component abstraction. The publisher does not need to know its subscribers, and subscribers do not need to know publisher implementations. Broker owns streams, authoritative message queues/logs, published messages, ordering, fan-out, routing, dispatch, subscriptions, delivery, retention and replay. Per-subscriber queues are bounded delivery buffers, not separate message histories.

Application is the client-facing/external application participant. Security and Admin are internally provisioned participants. Tracing participates internally and also has an external trace access/integration surface; both use the same Broker-owned stream data and existing authentication/authorisation model. Each participant's behaviour differs by domain responsibility, not by using separate messaging infrastructure.

The user has also clarified the relationship between Broker and Core: Broker owns its queues and published messages. Stream snapshots and persistence operate on that Broker-owned stream/queue/message state; Core provides the shared reusable snapshot/persistence mechanism and policy. There must be no duplicate authoritative stream store and no subsystem-specific queue/history/persistence implementation for messages already owned by Broker. Persistence restoration restores Broker-owned stream state before it is used. Existing stream fan-out semantics remain in force: every authorised subscriber receives its own delivery; this is not competing-consumer work-queue semantics.

This decision supersedes any description of direct component-to-component messaging, independent subsystem message buses, Tracing-owned stream history/queues, or Tracing-owned message persistence. Preserve domain-specific trace event schemas, authentication rules, delivery guarantees and correctness-specific gap handling, but express them through the common Broker stream model. Any remaining conflict between the new ownership model and older trace-specific cursor, sequence, file, snapshot or replay rules must be reconciled before implementation; do not silently preserve a second source of truth or invent a mapping.


The former “Internal subscribers and the Security role” discussion has been reconciled with the authoritative settled decisions below. Internal subscriber identity and provisioning, the existing Security authorisation request/decision contracts and validation, delivery and failure behaviour, timeout handling, tombstone retention, and recovery are settled; they are not open design topics.

The universal HTTP rate-limiting requirement, token-bucket mechanism, four endpoint categories and their default refill rates and bucket capacities are settled: public read/discovery 120 requests/minute with capacity 30; write/mutation 60 requests/minute with capacity 15; SSE establishment 60 requests/minute with capacity 10; and administration 30 requests/minute with capacity 10. Each category remains independently configurable; new endpoints use an existing category unless an explicit decision adds another. Identity/IP accounting, bounded bursts, standard `429` responses and trusted-proxy-only forwarded IP handling are also settled. No further rate-limit default decision is open. HTTP request rate limiting is independent of the already-settled SSE concurrent-connection limit. The retained message history is owned by Broker and shared across all authorised subscribers to the relevant stream; there is no subscriber-specific copy or Tracing-owned retention window. The older traceSeq/trace.gap cursor-boundary behaviour is not independently authoritative after the Broker-stream decision. Its compatibility with Broker's stream seq, Last-Event-ID, retained queue and retention-gap notification (removed) semantics must be reconciled in the public contract before implementation.

The former trace-specific JSON Lines writer, 50ms trace flush target, trace file rotation, atomic trace-file replacement and trace-file restoration decisions are superseded wherever they describe a Tracing-owned persistence subsystem. They are not Broker persistence requirements. Broker-owned stream state is snapshotted and persisted through the shared Core mechanism and the stream-class/per-stream persistence policy. Any Broker persistence guarantees or operational details not already settled in the Storage and Broker sections remain open; do not carry forward trace-specific file settings as defaults for Broker.

The central-Core stream persistence direction and its foundational scope are settled: persistence and snapshots are a shared Core capability reusable across pub/sub modules for `Application`, `System`, `Security` and `Admin`. The stream class is determined by the stream's role. Persistence policies are defined at stream-class level, with optional per-stream overrides for exceptions. A stream class with no explicit persistence policy is non-persistent by default: persistence requires an explicit class-level opt-in, while a per-stream override may opt an individual stream in or out. This decision does not make authentication/authorisation state, credentials, pending requests, sessions, or all Security/Admin control-plane events persistent; the existing in-memory-only authentication and authorisation state remains unchanged. Assess sensitive data, security boundaries, retention, replay and recovery when reviewing subsequent policy details. No implementation.

Any newly identified decision that changes scope, externally visible behaviour or an acceptance criterion must be discussed and explicitly agreed before it is added to **Settled decisions**. Do not infer or silently settle decisions.

## Next-session handoff

### Authoritative current architecture — read this first

The latest user-agreed macro-architecture is the governing direction for the next session and supersedes conflicting older subsystem-specific descriptions in this document:

1. **One Broker, one stream model.** Broker is the shared messaging infrastructure and the communication backbone for the whole system. Application, Security, Tracing and Admin are independent participants in this same infrastructure. Each participant may publish to and subscribe to streams according to its responsibility, authenticated identity, role and authorised scope. They are not merely subscribers.
2. **Streams are the only component-to-component communication mechanism.** A component publishes a message to a Broker stream; interested components subscribe to that stream and process it according to their own responsibility. Publishers do not know or call subscribers; subscribers do not depend on publisher implementations. No direct component-to-component method calls, RPC, private event channels or bespoke integration bus for domain communication. Ordinary dependency injection and calls to shared Core infrastructure are not domain messaging.
3. **Participant roles differ; infrastructure does not.** Application is the client-facing/external participant. Security and Admin are internally provisioned participants. Tracing participates internally and also has a documented external trace access/integration surface. External and internal access use the same Broker stream model and established authentication/authorisation rules; external transport adapters do not create a second messaging architecture. A participant may both publish and subscribe.
4. **Broker owns messaging state.** Broker owns stream registration and lifecycle, authoritative stream queues/logs and published messages, ordering, fan-out, routing, dispatch, subscriptions, bounded delivery buffers, delivery outcomes, retention, replay and gaps. Fan-out pub/sub applies: every authorised subscriber receives its own delivery; this is not competing-consumer work-queue semantics. Per-subscriber queues are bounded delivery buffers, not independent message histories.
5. **Core supplies shared snapshot/persistence capabilities.** Core provides reusable snapshot and persistence mechanisms and policy. Those mechanisms operate on Broker-owned stream and message state; they do not create a second authoritative message store. Every stream has its schema-governed in-memory snapshot. Persistence is selected by stream class with optional per-stream overrides and is non-persistent by default unless explicitly opted in. Restoring persisted stream data restores Broker's authoritative state before it is served. Security authentication/authorisation state remains non-persistent; plaintext bearer credentials are never retained in snapshots.
6. **Tracing is a normal Broker participant, not a separate recorder architecture.** It subscribes to relevant source streams and publishes trace events to Broker-owned tracing stream(s). Broker owns trace messages, queues, ordering, retention, replay and delivery. Core's shared snapshot/persistence policy applies to tracing streams. Tracing must not own a parallel per-stream history, queue, persistence worker, JSONL store, file rotation system, snapshot or restore path. The external trace API and monitor consume Broker-owned trace-stream data.
7. **Preserve domain correctness and security.** Security's strict continuity and fail-closed behaviour remain distinct from Tracing's observational behaviour. Existing identity provisioning, authorisation workflows, validation, deadlines, tombstones, credential handling, rate limits, stream-scope rules and other agreed domain contracts remain in force unless they conflict with the architecture above. Broker transports messages; it does not make Security policy decisions.

### Explicitly superseded historical Tracing decisions

Earlier parts of this file record a memory-first Tracing design with a Tracing-owned bounded history, traceSeq replay, trace.gap, asynchronous JSON Lines persistence, 50ms flush target, coalesced dirty scheduling and atomic trace-file snapshots. Those decisions were made before the shared Broker ownership model was clarified. They are **not current implementation requirements** wherever they imply a separate Tracing queue/history, persistence mechanism, replay source or cursor authority. Do not revive them merely because they remain in historical sections or old settled-decision bullets.

Preserve the meaning and privacy requirements of TraceEvent and the agreed observational/reconnection behaviour where compatible. However, reconcile TraceEvent.traceSeq, Last-Event-ID, trace.gap, source-stream seq, Broker stream seq, retention-gap notification (removed), trace API addressing and monitor behaviour with Broker's authoritative stream envelope and cursor semantics. Do not silently invent a mapping or retain a second sequence, retention, replay or persistence system. The former 50ms trace flush target, trace.jsonl layout, trace file rotation and trace-file atomic replacement rules are not Broker persistence requirements.

### Where to resume

**Settled decision — snapshot interval scope across stream classes:** The user approved class-specific fixed snapshot intervals. Each persistence-enabled stream class defines its own interval because Application, System and Admin may have different recovery and write-frequency requirements. Security authentication/authorisation state remains non-persistent and has no durable snapshot interval. Individual streams may still have explicit fixed overrides. Core continues to use one global scheduler to evaluate each stream against its effective interval; class-specific intervals do not create per-stream timers.

The Application stream snapshot interval is approved at a fixed 5 seconds, the System stream snapshot interval at a fixed 1 second, and the Admin stream snapshot interval at a fixed 10 seconds. The 15-second crash-loss target is also settled under Snapshot scheduling. Do not reopen these decisions without explicit user direction.

The broader macro-architecture has been clarified, and the event-only messaging model is now settled. The Admin/Security workflow and stream-class responsibilities have been reviewed and recorded below. The user has approved the Broker stream registration/lifecycle contract: schema registration is idempotent by canonical schema hash; re-registration returns the existing stream without changing history, sequence, retention or subscriptions; ordinary stream deletion and reset are not supported in the initial version; any future retirement or migration requires an explicit design. The `schema.registered` event is emitted only for new schema admission. The user has also approved schema immutability: a registered schema cannot be changed in place; a changed schema hash creates a distinct stream while the old stream remains registered, and schema-version transitions require explicit application-level migration. The next review area is **Admin/Security workflow and orchestration**, including the responsibilities and boundaries of their stream classes and the sequence of request, Security decision and Admin action. Review one architectural decision at a time. Revisit Broker mechanics only where needed to support these workflows, without moving policy or orchestration ownership into Broker. **Settled decision — subscription identity and duplicate subscriptions:** every successful registration creates a new unique Broker-generated `subscriptionId`; the same subscriber may hold multiple independent simultaneous subscriptions to the same stream; each has its own lifecycle and bounded delivery buffer; overflow or closure affects only that subscription; reconnect attaches anew and resumes from the client-supplied `Last-Event-ID` against authoritative Broker history; Security revocation terminates all subscriptions for the revoked subscriber–stream relationship. See the Broker subscription identity section for the complete contract.

**Settled decision — universal event-only messaging model:** Pulsyflux uses events as its universal messaging concept. Every publication publishes an event carrying message data; every subscription listens for events and consumes their data. An event may express a request for an action, a decision, a notification or a fact; these are meanings interpreted by the receiving participant, not distinct command/event message categories. There is no separate command message kind or command bus. All participants use the same Broker streams and generic messaging infrastructure. Broker does not interpret event meaning, validate domain authorisation, understand stream classes or orchestrate participant workflows. Each participant interprets and handles the events relevant to its own responsibility. Preserve the settled Security request/decision/registration contracts, schemas, validation, deadlines, timeout handling, tombstones, recovery, credential rules and fail-closed behaviour.

Before proposing anything, read this handoff, the complete **Settled decisions** section, the complete **Open decisions** section, and the relevant Broker and subsystem sections. Treat this handoff's architecture as the current macro-level direction while preserving compatible settled domain decisions. Where an older settled bullet conflicts with the new ownership model, flag the conflict and reconcile it explicitly; do not silently apply both architectures.

**Discussion rules:** Architecture/design only. Do not implement, scaffold, create code, or change other project files. Discuss exactly one architectural decision at a time. For each proposal, explain both subsystem-specific behaviour and macro-architectural implications. A proposal remains open until the user explicitly agrees. After each agreement, update this file directly on `main`, preserve compatible prior decisions, clearly supersede conflicting obsolete wording, then fetch the committed file again to verify the change.

**Last completed work:** In the current session, the user agreed that the retention notification should be named retention-gap notification (removed), not stream.gap, because it is a Broker-generated control event rather than an event in the stream. Its settled semantics are: Broker-generated SSE control event; not appended to stream history or persisted; no Broker stream sequence and no SSE id; sent when the resume cursor predates the earliest retained message, followed by replay from earliestAvailableSeq; payload fields streamId, requestedAfterSeq, earliestAvailableSeq. The user then raised the strict per-stream JSON Schema constraint: external clients must not receive control data as though it were ordinary stream-message data that must conform to the stream's schema. The assistant proposed separate Broker-owned schemas for control events; this is not yet approved. Exact SSE wire encoding is also open. The next session must resume with the separate-control-schema question, one decision at a time.

**Earlier completed work:** The user confirmed the event-only messaging model: every publication publishes an event carrying message data, and every subscription listens for events and consumes their data. Events may express requests, decisions, notifications or facts; these are interpreted by participants and are not separate command/event message kinds. There is no separate command message kind or command bus. Broker is shared infrastructure underneath all Application, Security, Admin and Tracing stream communication; all publishers and subscribers use it. Broker is completely unaware of domain stream classes and workflows and does not interpret event meaning, validate domain authorisation, check publisher roles or orchestrate registration. Security owns authorisation; Admin owns and initiates registration after positive authorisation using Broker's generic subscription-management capability; Broker performs generic messaging and subscription mechanics only. The command/event distinction is settled; the next session should read the current Open decisions and choose the next unresolved architectural decision, one at a time. The user previously approved that incoming client-facing subscription/access requests use the Application stream class. This was recorded in Settled decisions. The user also approved the Admin/Security/Broker subscription-registration responsibility boundary: Security makes the authorisation decision; Admin waits for the positive outcome and initiates registration; Broker creates the runtime subscription and unique `subscriptionId` and manages delivery/lifecycle, but does not make policy decisions or hold pending administrative registrations. This was committed directly to `main` and verified. The user also approved Broker subscription identity and duplicate-subscription semantics. Every successful registration creates a unique Broker-generated `subscriptionId`; repeated registration by the same subscriber creates an independent subscription, each with its own lifecycle and bounded delivery buffer; reconnect attaches anew and resumes from `Last-Event-ID`; Security revocation terminates all subscriptions for the affected subscriber–stream relationship. This decision was committed directly to `main` and verified. The user previously approved Broker schema immutability: changed canonical schema hashes create distinct streams, and schema-version transitions require explicit application-level migration. The shared Broker/Core/participant architecture was added to the mission, foundational architecture, Broker semantics introduction, Storage, Tracing, package-layout guidance, Open decisions and this handoff. Tracing was reframed as a Broker participant. Older trace-specific persistence and cursor decisions were marked for supersession/reconciliation. The Broker delivery, retention, replay, persistence and snapshot decisions listed below have also been approved and recorded. The user approved fixed, non-adaptive, class-specific snapshot intervals, with optional per-stream overrides and one global scheduler. The Application class interval is approved at 5 seconds, the System class interval at 1 second, and the Admin class interval at 10 seconds; the 15-second crash-loss target is settled, with its operational limitations recorded under Snapshot scheduling. The user also approved idempotent schema registration and no ordinary stream deletion or reset in the initial version; see the Broker stream registration and lifecycle contract. Always fetch the current `main` branch before continuing; do not rely on commit identifiers from older handoffs.

**Settled decision — snapshot interval configuration and behaviour:** The interval is part of the policy provided by each stream class. Each persistence-enabled stream uses its class's configured snapshot interval unless an explicit per-stream override is supplied. Core retains one global periodic scheduler; it evaluates each eligible stream against that stream's effective interval and snapshots only changed persisted state when due. Per-stream overrides do not create per-stream timers. Intervals are fixed configured values: Core does not adapt them automatically to publication activity or observed load. The Admin class interval is approved at 10 seconds.

**Persistence decisions:** The snapshot boundary, atomic-promotion mechanisms, fixed class intervals and 15-second crash-loss target are settled as described in this file. The 15-second value is an operational target under healthy persistence conditions, not an unconditional guarantee.

**Superseded resume instruction:** The earlier direction to address the bounded crash-loss window and continue the Broker semantics review is complete or superseded by the settled decisions recorded above. Resume with the Admin/Security workflow and orchestration review, beginning with their stream-class responsibilities. Keep the work architecture/design only and preserve the shared Broker stream model.


## Session handoff — Broker retention limits

The user has approved the three-dimensional retention model and oldest-first eviction ordering. The settled rules are recorded under **Settled Broker retention model and eviction ordering** in Open decisions:

- Maximum age, maximum retained message count and maximum retained storage size are independent, cumulative hard limits.
- Limits are configured by stream class, with optional per-stream overrides.
- Age is measured from Broker acceptance time.
- Expired messages are eligible for eviction; count or storage-size excess triggers oldest-first eviction.
- When any limit is exceeded, evict oldest retained messages until all configured limits are satisfied.
- Enforcement applies to Broker's authoritative stream history, never subscriber delivery buffers.
- The illustrative values 7 days, 100,000 messages and 500 MB are not approved defaults.

**Next-session starting point — resume exactly here:** continue the Broker public-contract discussion at Broker-generated control-event schemas and stream-schema isolation. The user has just pointed out that each stream has its own strict JSON Schema and an external client must not receive Broker control data as though it were a stream message that should conform to that schema. The assistant proposed that Broker control events have their own explicitly defined schemas, separate from all stream schemas; this proposal is awaiting the user's decision. Ask/discuss only this one question first. Do not assume approval. After it is settled, discuss the exact SSE wire representation of retention-gap notification (removed) (the suggested event: retention-gap notification (removed) plus JSON data: and no id: is not yet approved). Preserve all settled event-only messaging and Broker ownership decisions, especially Broker's complete ignorance of stream classes and domain workflows. No implementation.

**Earlier Broker decisions already settled:** publication acceptance versus crash durability: successful publication is in-memory acceptance, opted-in snapshots may be persisted asynchronously, and messages newer than the latest completed durable snapshot may be lost after abrupt failure. Subscriber delivery-buffer overflow and delivery confirmation/resume-cursor ownership are now settled: detach only the affected full subscription; do not block publication or other subscribers; do not silently drop messages; preserve authoritative history; and allow reconnect/replay from the last sequence the client received and supplied through `Last-Event-ID`; if retention has overtaken the cursor, replay begins with the earliest event still retained and evicted events are unavailable. Delivery confirmation is also settled: each SSE event ID carries the Broker stream sequence; the client-supplied `Last-Event-ID` is the resume cursor; server write/flush success is not proof of client receipt or processing; and no explicit acknowledgement protocol is added. All approved retention decisions are recorded above: three cumulative hard limits; oldest-first eviction; hybrid enforcement on each accepted publication plus one global age-expiry pass every minute; canonical serialized-byte accounting including the complete Broker envelope and stored metadata; a separate per-stream-class individual-message maximum; oversized publication rejection before append without eviction or sequence consumption; and explicit retention-gap notification (removed) reporting when a resume cursor predates the earliest retained message. If a resume cursor predates the earliest retained event, replay begins at the earliest retained event; evicted events are unavailable and no separate gap notification is sent. Without a resume cursor, replay begins at the earliest retained event.

**Settled decision — publication acceptance and crash durability:** a successful publication is in-memory acceptance, not durable persistence; opted-in snapshots may be persisted asynchronously without blocking publication; after abrupt failure, messages newer than the latest completed durable snapshot may be lost; restart restores the latest available persisted snapshot before normal stream use; non-opted-in streams and Security authentication/authorisation state remain non-persistent. The class snapshot intervals are settled (Application 5 seconds, System 1 second, Admin 10 seconds), as is the 15-second crash-loss target under healthy persistence conditions. Stream-class interval policy with optional per-stream overrides, the global scheduling, per-stream snapshot boundary and atomic-promotion models are settled separately.

**Settled decision — delivery confirmation and resume-cursor ownership:** each SSE event ID carries the Broker stream sequence; clients resume using their supplied `Last-Event-ID`; server write/flush success is not treated as proof of client receipt or processing; no explicit acknowledgement protocol or subscriber-owned message history is introduced. Existing replay and ordinary replay semantics apply, while Security's strict continuity and fail-closed requirements remain in force.

**Settled decision — subscriber delivery-buffer overflow:** the user approved detaching only the subscription whose bounded delivery buffer becomes full. Broker does not block publication or other subscribers, does not silently drop messages, and leaves authoritative history unchanged. The detached subscriber may reconnect and resume from its last successfully delivered sequence; if retention has overtaken that cursor, Broker reports the standard ordinary replay behaviour and resumes from the earliest retained message. This policy applies uniformly across participants while preserving domain-specific correctness requirements, including Security's strict continuity and fail-closed behaviour.

### Settled Broker/Core delivery, retention and persistence decisions

**Subscriber delivery-buffer overflow**
- Every subscriber has an independent bounded temporary delivery buffer. If one becomes full, Broker detaches only that subscription.
- A full buffer never blocks stream publication or delivery to other subscribers. Broker never silently drops messages to make room.
- Detachment does not alter authoritative history. The subscriber may reconnect and replay from its last received sequence using Broker history. If retention has overtaken its cursor, Broker resumes from the earliest event still retained; evicted events are unavailable and no separate gap notification is sent.
- Preserve domain-specific correctness, including Security's strict continuity and fail-closed requirements.

**Delivery confirmation and resume cursor**
- Each SSE event ID carries the Broker stream sequence. The client-supplied `Last-Event-ID` is the authoritative resume cursor.
- Server-side write/flush success is only a transport observation, not proof of client receipt, acknowledgement or processing.
- No separate per-message acknowledgement protocol or subscriber-owned message history is introduced.

**Retention and replay**
- Three independent cumulative hard limits apply to each authoritative stream history: maximum age, maximum retained message count and maximum retained storage size. They are configured by stream class with optional per-stream overrides; age is measured from Broker acceptance time.
- Eviction is oldest-first and continues until all configured limits are satisfied. Retention applies to Broker history, not subscriber delivery buffers.
- Enforce retention on every accepted publication and with one global age-expiry maintenance pass every minute, including idle streams.
- Storage-size accounting uses canonical serialized bytes including the complete Broker envelope and stored metadata. A separate per-stream-class individual-message maximum applies.
- Reject an oversized publication before append, without eviction or sequence consumption.
- Replay uses authoritative retained history. If a stale resume cursor predates the earliest retained event, replay begins at the earliest retained event; evicted events are unavailable and no separate gap notification is sent. Without a resume cursor, replay begins at the earliest retained event.
- Illustrative values 7 days, 100,000 messages and 500 MB were not approved defaults.

**Publication acceptance, snapshots and persistence**
- Successful publication means in-memory acceptance and append to Broker's authoritative history, not durable persistence. Publication does not wait for snapshots.
- Persistence is opt-in; non-opted-in streams remain non-persistent. Security authentication/authorisation state remains non-persistent, and plaintext bearer credentials are never snapshotted.
- Core owns shared snapshot/persistence capabilities over Broker-owned state; no second authoritative message store is created. On restart, restore the latest successfully persisted snapshot before normal stream use. Sequence continues from the last restored message under existing sequence/`runId` rules; a stream without restored messages starts at sequence 1 and a new process receives a new `runId`.
- One global periodic Core scheduler checks persistence-enabled streams and snapshots only streams whose persisted state changed since their last successfully completed snapshot. There are no per-stream timers and no synchronous per-publication snapshots.
- Each snapshot is a consistent point-in-time state of one stream, identified by its highest included Broker sequence. Messages accepted after that boundary are excluded from the snapshot in progress and remain in authoritative history for a later snapshot. Publication is not blocked.
- Snapshot completion requires the complete captured state to be persisted. Write a new snapshot separately, validate it, then atomically promote it as latest completed snapshot; retain the previous completed snapshot until promotion succeeds. On write, validation or promotion failure, preserve the previous completed snapshot as recovery point.
- After abrupt process failure, messages accepted after the latest successfully completed durable snapshot may be lost. The approved 15-second crash-loss target is specified under Snapshot scheduling; it is not an unconditional guarantee.
- Snapshot interval configuration and behaviour are settled: stream-class policy supplies a fixed interval, with an optional fixed per-stream override; one global scheduler evaluates each stream against its effective interval. Automatic activity/load-based interval adaptation is not used. Actual interval values remain unresolved.

### Current architectural decisions carried forward

- Broker owns each stream's authoritative ordered message history and assigns one monotonically increasing sequence per stream when a message is accepted.
- Rejected publications do not consume sequence numbers. Fan-out deliveries retain the same stream sequence.
- Sequence continues from the last restored message after restart; a new process receives a new `runId`. A stream with no restored messages starts at sequence 1.
- Broker owns retention of authoritative history. Replay reads from that history; subscriber delivery buffers are bounded, temporary and independent.
- Core provides shared snapshot and persistence mechanisms for Broker-owned state. Snapshots capture defined Broker state; persistence durably stores selected snapshot data; restoration repopulates Broker's authoritative state before normal use.
- Core uses one global periodic scheduler for persistence-enabled streams and snapshots only changed persisted state since the last successfully completed snapshot; there are no per-stream timers or synchronous per-publication snapshots. Each stream class supplies a fixed snapshot interval as part of its policy, with an optional fixed per-stream override. Core does not automatically adapt intervals to publication activity or observed load; the Admin class interval is 10 seconds.
- Actual snapshot interval values and the 15-second crash-loss target are settled. The global scheduling model, per-stream point-in-time snapshot boundary, atomic promotion, acceptance-versus-durability distinction and possible loss after the latest completed snapshot are also settled.
- Work remains architecture/design only: no implementation or scaffolding. Continue editing `steering.md` on `main`, preserving previously agreed decisions.
 a Broker retention-gap event identifies the affected stream with `streamId`, the subscriber's supplied resume cursor with `requestedAfterSeq`, and the first sequence still available for replay with `earliestAvailableSeq`. The missing range is derived as `requestedAfterSeq + 1` through `earliestAvailableSeq - 1`; it is not duplicated as separate fields. The event reports the retention gap and does not alter stream history or message sequencing. This settles the payload fields; exact wire encoding remains a separate public-contract decision. The control event's unsequenced, non-persistent status is settled separately above.\n\n**Settled decision — subscriber delivery-buffer overflow:** the user approved detaching only the subscription whose bounded delivery buffer becomes full. Broker does not block publication or other subscribers, does not silently drop messages, and leaves authoritative history unchanged. The detached subscriber may reconnect and resume from its last successfully delivered sequence; if retention has overtaken that cursor, Broker reports the standard ordinary replay behaviour and resumes from the earliest retained message. This policy applies uniformly across participants while preserving domain-specific correctness requirements, including Security's strict continuity and fail-closed behaviour.

### Settled Broker/Core delivery, retention and persistence decisions

**Subscriber delivery-buffer overflow**
- Every subscriber has an independent bounded temporary delivery buffer. If one becomes full, Broker detaches only that subscription.
- A full buffer never blocks stream publication or delivery to other subscribers. Broker never silently drops messages to make room.
- Detachment does not alter authoritative history. The subscriber may reconnect and replay from its last received sequence using Broker history. If retention has overtaken its cursor, Broker resumes from the earliest event still retained; evicted events are unavailable and no separate gap notification is sent.
- Preserve domain-specific correctness, including Security's strict continuity and fail-closed requirements.

**Delivery confirmation and resume cursor**
- Each SSE event ID carries the Broker stream sequence. The client-supplied `Last-Event-ID` is the authoritative resume cursor.
- Server-side write/flush success is only a transport observation, not proof of client receipt, acknowledgement or processing.
- No separate per-message acknowledgement protocol or subscriber-owned message history is introduced.

**Retention and replay**
- Three independent cumulative hard limits apply to each authoritative stream history: maximum age, maximum retained message count and maximum retained storage size. They are configured by stream class with optional per-stream overrides; age is measured from Broker acceptance time.
- Eviction is oldest-first and continues until all configured limits are satisfied. Retention applies to Broker history, not subscriber delivery buffers.
- Enforce retention on every accepted publication and with one global age-expiry maintenance pass every minute, including idle streams.
- Storage-size accounting uses canonical serialized bytes including the complete Broker envelope and stored metadata. A separate per-stream-class individual-message maximum applies.
- Reject an oversized publication before append, without eviction or sequence consumption.
- Replay uses authoritative retained history. If a stale resume cursor predates the earliest retained event, replay begins at the earliest retained event; evicted events are unavailable and no separate gap notification is sent. Without a resume cursor, replay begins at the earliest retained event.
- Illustrative values 7 days, 100,000 messages and 500 MB were not approved defaults.

**Publication acceptance, snapshots and persistence**
- Successful publication means in-memory acceptance and append to Broker's authoritative history, not durable persistence. Publication does not wait for snapshots.
- Persistence is opt-in; non-opted-in streams remain non-persistent. Security authentication/authorisation state remains non-persistent, and plaintext bearer credentials are never snapshotted.
- Core owns shared snapshot/persistence capabilities over Broker-owned state; no second authoritative message store is created. On restart, restore the latest successfully persisted snapshot before normal stream use. Sequence continues from the last restored message under existing sequence/`runId` rules; a stream without restored messages starts at sequence 1 and a new process receives a new `runId`.
- One global periodic Core scheduler checks persistence-enabled streams and snapshots only streams whose persisted state changed since their last successfully completed snapshot. There are no per-stream timers and no synchronous per-publication snapshots.
- Each snapshot is a consistent point-in-time state of one stream, identified by its highest included Broker sequence. Messages accepted after that boundary are excluded from the snapshot in progress and remain in authoritative history for a later snapshot. Publication is not blocked.
- Snapshot completion requires the complete captured state to be persisted. Write a new snapshot separately, validate it, then atomically promote it as latest completed snapshot; retain the previous completed snapshot until promotion succeeds. On write, validation or promotion failure, preserve the previous completed snapshot as recovery point.
- After abrupt process failure, messages accepted after the latest successfully completed durable snapshot may be lost. The approved 15-second crash-loss target is specified under Snapshot scheduling; it is not an unconditional guarantee.
- Snapshot interval configuration and behaviour are settled: stream-class policy supplies a fixed interval, with an optional fixed per-stream override; one global scheduler evaluates each stream against its effective interval. Automatic activity/load-based interval adaptation is not used. Actual interval values remain unresolved.

### Current architectural decisions carried forward

- Broker owns each stream's authoritative ordered message history and assigns one monotonically increasing sequence per stream when a message is accepted.
- Rejected publications do not consume sequence numbers. Fan-out deliveries retain the same stream sequence.
- Sequence continues from the last restored message after restart; a new process receives a new `runId`. A stream with no restored messages starts at sequence 1.
- Broker owns retention of authoritative history. Replay reads from that history; subscriber delivery buffers are bounded, temporary and independent.
- Core provides shared snapshot and persistence mechanisms for Broker-owned state. Snapshots capture defined Broker state; persistence durably stores selected snapshot data; restoration repopulates Broker's authoritative state before normal use.
- Core uses one global periodic scheduler for persistence-enabled streams and snapshots only changed persisted state since the last successfully completed snapshot; there are no per-stream timers or synchronous per-publication snapshots. Each stream class supplies a fixed snapshot interval as part of its policy, with an optional fixed per-stream override. Core does not automatically adapt intervals to publication activity or observed load; the Admin class interval is 10 seconds.
- Actual snapshot interval values and the 15-second crash-loss target are settled. The global scheduling model, per-stream point-in-time snapshot boundary, atomic promotion, acceptance-versus-durability distinction and possible loss after the latest completed snapshot are also settled.
- Work remains architecture/design only: no implementation or scaffolding. Continue editing `steering.md` on `main`, preserving previously agreed decisions.
