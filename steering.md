
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

Build one general-purpose message bus in Go: publishers, subscribers and a broker queue that serves them. Each stream is identified by the hash of a JSON Schema. The whole public surface is generated from a single AsyncAPI document. Applications that need messaging use this bus instead of building their own.

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

In-memory only for the first release, bounded by retention limits.

All streams and retained messages are lost when the process exits.

Authentication state is also in-memory only. Pulsyflux does not create authentication, approval, subscriber-record or credential files. Pending requests, approved Application relationships, System and Security credentials, the single Admin credential record, credential hashes and active connections are lost when the process exits.

Durable storage for streams and messages is out of scope unless separately specified.

### Docs UI

Served at `/`, with a "Try it" console generated from the contract.

### Tracing

Tracing is enabled by default.

Publish and subscription events for known streams are recorded per stream in a local JSON Lines file and exposed through the public trace API. The built-in `/streams` HTML viewer is an optional convenience client of that same API.

A file-sink failure is reported by `/health`; it does not stop the bus.

### Monitor UI

Served at `/streams`: an optional, built-in HTML trace viewer for users who do not want to build their own client. It lists streams and displays the live chronological trace by consuming the public `GET /v1/streams/{streamId}/trace` SSE endpoint. It is a convenience UI, not a separate tracing capability or data source.

### Authentication administration UI

An HTML administration page is served locally by Pulsyflux. It is an Admin-role subscriber, not a separate authentication client or privileged exception.

The administrator presents the configured `PULSYFLUX_ADMIN_TOKEN` as its bearer credential. Pulsyflux verifies it through the same hash-and-lookup mechanism used for every subscriber. The matched credential record supplies the global Admin role. No administrator browser session or second authentication credential is created.

After authentication, the page discovers available streams and lets the administrator select a stream to review pending Application subscriber requests and approved relationships. The selected stream scopes an approval/revocation operation, not the Admin credential.

The interface is part of the existing Pulsyflux service. It is not a separate application or service.

### Test libraries

Standard `testing` package only.

### Authentication

Authentication is built into Pulsyflux and uses one shared in-memory credential store and one bearer-token verification mechanism for all subscriber roles. The four roles are `Application`, `System`, `Security` and `Admin`.

There is exactly one authentication and authorisation model throughout Pulsyflux: the standard subscriber model used by the internal Security subscriber and by ordinary publish and subscribe operations. Every operation that requires an authenticated identity or an authorisation decision must use this same model, shared credential store, credential verification, and identity/role/scope-based enforcement. HTTP routes, browser interfaces, administrative operations, schema retrieval, publishing, subscribing and tracing are not permitted to introduce a separate authentication mechanism, parallel authorisation framework, privileged bypass, or special-case trust path. The read-only `GET /v1/streams` discovery operation and `GET /v1/streams/{streamId}/schema` schema retrieval operation are explicitly public and do not require subscriber authentication; this grants no access to privileged operations. Both remain subject to the universal API rate-limiting requirement. Internal Security, System and Admin identities are represented within this same model; their different capabilities arise only from their trusted identity, role and scope. The Security bootstrap described below is the sole provisioning step needed to establish its reserved identity and credential in the shared store; it does not create an alternative authentication or authorisation model.

- **Application:** an external subscriber approved for one Subscriber ID + Stream ID relationship.
- **System:** a Pulsyflux-managed internal subscriber to a stream, such as Tracing. Pulsyflux creates and manages its identity and credential; external subscribers cannot claim this role.
- **Security:** a reserved Pulsyflux-managed internal subscriber role for authentication and authorisation. Pulsyflux creates and manages its identity and credential at startup; external subscribers cannot request, claim, approve or impersonate this role.
- **Admin:** the single global administrator role. It is not scoped to a stream and can administer every stream.

Every subscriber presents a bearer token. Pulsyflux hashes it, finds the matching in-memory credential record, validates it and obtains the subscriber identity, role and stream scope. Authentication identifies the subscriber; authorisation checks the record's role and scope against the requested operation. A client cannot choose or elevate its own role.

The credential store contains hashes, not plaintext tokens. Application, System and Security records are scoped to their relevant subscriber–stream relationship. The Admin record is global and has no stream scope.

At startup, before the Security subscriber starts, the core generates a fresh 256-bit random bearer credential for its reserved Security identity. It seeds the shared in-memory credential store with the credential hash and the trusted Security identity, role and scope. The core passes the plaintext credential directly to the Security subscriber through a private in-memory startup handoff; the credential is not exposed through an API, event, log, trace or external interface. The handoff is one-time and is not a second subscription or publishing path.

The Security subscriber then authenticates and subscribes through the ordinary bearer-token verification and subscription mechanism. It publishes through the ordinary publishing mechanism. The core's generic credential verification resolves the seeded Security record without asking the Security subscriber to authorise its own authentication or subscription. This is the minimal bootstrap trust check; it does not create a separate messaging mechanism or allow a client to select the Security role.

Exactly one Admin credential record must exist in every running instance. Pulsyflux creates it at startup from the required `PULSYFLUX_ADMIN_TOKEN` environment variable. The token must contain at least 256 bits of cryptographically secure randomness. Pulsyflux hashes it and does not retain the plaintext. If the variable is missing or empty, Pulsyflux must fail startup. The Admin record cannot be requested, approved, duplicated or revoked through subscriber workflows. Changing the token requires restarting Pulsyflux.

The HTML administration page is an Admin subscriber. It authenticates by presenting the Admin bearer token through the same hash lookup and role resolution as every other subscriber. There is no separate administrator login protocol, browser session or authentication bypass. Admin authority is global; selecting a stream in the UI only selects the target of an operation.

Application subscribers identify themselves with a stable Subscriber ID (a UUID generated by the subscriber installation and reused across restarts) and a human-readable Name. The Name is descriptive only, maintained per subscriber–stream relationship, and is not a credential.

A new Application Subscriber ID + Stream ID relationship enters the pending-approval workflow. The subscriber attempts to subscribe and supplies its Subscriber ID and Name. If no approved relationship exists, Pulsyflux records a pending request in memory and holds the SSE connection open for up to 10 minutes while awaiting Admin approval. If no approval is received within that window, the pending request expires and is no longer eligible for approval; this expiry is not an explicit denial. A decision arriving after expiry cannot approve or otherwise change the request. The 10-minute limit is distinct from the 5-second default timeout for ordinary Security authorisation responses. For a request awaiting Admin approval, the original correlation remains eligible for the administrator instruction and resulting Security decision only during that 10-minute window; the ordinary 5-second timeout does not expire it. No message events flow before approval. The administrator selects a stream in the Admin UI and explicitly approves or rejects the request.

On approval, Pulsyflux creates an Application credential record for that exact relationship, retains only its hash in memory, delivers the plaintext credential once on the existing pending SSE connection using the `pulsyflux.credential` event, and allows message events to flow. The relationship remains authorised for the lifetime of the running process. Reconnecting with its bearer credential does not require another approval, and approval for one stream never grants access to another.

Pulsyflux generates Application, System and Security bearer credentials using Go's `crypto/rand`: 32 random bytes (256 bits), encoded with unpadded Base64 URL-safe encoding (`base64.RawURLEncoding`). Only the SHA-256 digest is retained. Verification hashes the presented token and compares digests using a constant-time comparison. Returning subscribers present credentials only through the HTTP `Authorization: Bearer <credential>` header. Credentials must never appear in URLs, query parameters, traces, ordinary logs, errors or the administration interface. Credential delivery is one-time, uses cache-prevention headers and is excluded from SSE replay.

Explicit rejection and expiry of the 10-minute pending-approval window close the pending SSE connection with the same generic `subscription.closed` event and `{"code":"not_authorised"}` payload; the subscriber cannot distinguish the outcome. Expiry is not recorded as an explicit denial. If approval was accepted before the window expired, a later SSE inactivity timeout may close the connection but does not reverse, revoke or otherwise change the already-established authorisation. An approved pending subscription proceeds on its existing SSE connection.

An authenticated Admin may revoke any approved Application Subscriber ID + Stream ID relationship. Revocation immediately invalidates its credential and closes active SSE connections for that relationship. If a subscriber loses its credential during the instance lifetime, the administrator revokes the relationship and the subscriber requests approval again for a new credential. An Application relationship may have at most one active SSE connection; a second simultaneous attempt receives HTTP `429 Too Many Requests`, leaving the existing connection open.

Subscriber IDs are normalised to lowercase. Names are required, trimmed, limited to 64 Unicode characters and rejected if they contain control characters. For an approved relationship, a Name update requires its valid bearer credential. A repeat pending request updates the existing Name without creating a duplicate or resetting the original 10-minute deadline. Concurrent updates are serialised per stream; the last processed update wins.

All authentication and authorisation state is in memory only: pending requests, Application, System and Security credential hashes, the single Admin record, role/scope records, active connections and revocation state. Pulsyflux creates, reads and writes no authentication-related files. On process exit, all this state disappears. On restart, Pulsyflux recreates exactly one Admin record from the environment variable; Application subscribers must request approval again, and System and Security subscribers are recreated internally.

Authentication is programming-language independent and uses documented HTTP and SSE interfaces. A Go client library is optional and never required.

## Why a custom generator

`lerenn/asyncapi-codegen` is the main Go AsyncAPI generator. It supports AsyncAPI 2.6.0 and 3.0.0, not 3.1.0. It generates broker controllers for Kafka and NATS, and does not generate an HTTP or SSE server.

No AsyncAPI renderer offers a Swagger-style "Try it out". The console has to be generated from the same contract to avoid drift.

The generator is a small Go program in the repo. It runs through `go generate`, commits its output, writes a fingerprint of the contract bytes, and has a check mode that fails when the committed output is not current.

The details are in Contract-first rules.

## Local-only rules

These rules apply to the running service. Development tooling (GitHub for source and CI, the Go module proxy, toolchain downloads) is not affected.

- The service listens only on a loopback address. The default is `127.0.0.1`, and the configured listen address must be a loopback IP (`127.0.0.0/8` or `::1`). A non-loopback address is a startup error, not a warning.
- The `servers` entry in `api/asyncapi.yaml` is `localhost` only. `contractgen` rejects any other host.
- The service makes no outbound network connections. That means no cloud storage or database, no hosted messaging, no telemetry export, no remote secret store, and no fetching of schemas, `$ref` targets or docs assets. Loopback connections used by local clients and integration tests are inbound service connections, not outbound dependencies.
- Everything the service needs at runtime is embedded in the binary, read from local disk, or supplied through the explicitly documented administrator environment variable. Authentication state itself is held only in memory; no authentication records or credentials are read from or written to files.
- Configuration comes from a local file and command-line flags, with the administrator token supplied only through the `PULSYFLUX_ADMIN_TOKEN` environment variable.
- The docs page loads nothing from a CDN or any other remote origin. The page sends a `Content-Security-Policy` header that allows only `'self'`.
- Because browsers on the same machine can reach `127.0.0.1`, the server rejects requests whose `Host` header is not a loopback name or address. This blocks DNS-rebinding attacks from web pages the user visits. For the same reason, it rejects cross-origin `Origin` headers on state-changing requests.
- The authentication administration page follows the same local-only and same-origin security restrictions.

`contractgen -check-architecture` enforces the no-outbound rule.

It fails if production code (outside `_test.go` and `internal/testsupport`) uses:

- `http.Client`
- `http.Get`
- `http.Post`
- `net.Dial*`
- `net.Resolver`

It also fails if production code imports `internal/testsupport` or `net/http/httptest`.

## Contract-first rules

### Workflow

1. Change `api/asyncapi.yaml`.
2. Run `go generate ./...`. Commit the generated output with the contract change.
3. Fix the compile errors by implementing or changing the hand-written code.
4. Run `make build`.

Never edit `generated/` by hand.

Never add a route, wire type or SSE frame in hand-written code.

### What `cmd/contractgen` does

`contractgen` reads only `api/asyncapi.yaml`.

It:

1. Validates the document against the official AsyncAPI 3.1.0 meta-schema from `asyncapi/spec-json-schemas`, embedded unchanged with `//go:embed`. An invalid document fails generation and lists every problem.
2. Enforces repo rules on the document:
   - Every server uses `protocol: http`.
   - Every `receive` operation is marked `x-transport: sse`.
   - Every message has at least one example, and each example validates against its payload schema.
   - Every payload schema in the AsyncAPI contract uses only the generator-supported keyword subset: `type`, `properties`, `required`, `items`, `enum`, `$ref`, `additionalProperties`, `format`, `pattern`, `minimum`, `maximum`, `minLength`, `maxLength`.
   - An unsupported keyword fails generation, so the contract cannot promise validation that the generated server does not perform.

This subset applies to schemas embedded in the AsyncAPI contract. It does not narrow the separate JSON Schema 2020-12 admission rules for user-registered stream schemas.

3. Writes these files into `generated/api/`:

#### `types.gen.go`

One Go type per message payload and component schema, with JSON tags.

#### `handler.gen.go`

The `Handler` interface: one method per operation, using only generated types.

`receive` operations take a generated `EventSink` that the method writes events to.

#### `server.gen.go`

`NewServer(h Handler, opts ServerOptions) http.Handler`.

It registers every route, decodes and validates requests against the contract schemas, maps typed errors to the documented responses, and writes SSE frames.

#### `console.gen.go`

The "Try it" console data:

- One form per `send` operation, prefilled from the message examples.
- One live viewer per `receive` operation.

#### `contract_fingerprint.go`

`const ContractSHA256`, the SHA-256 of the exact `api/asyncapi.yaml` bytes.

All output is formatted with `go/format`.

Output is deterministic: the same input always gives the same bytes.

`contractgen` also has check modes used by `make verify`:

- `-check-generated`
- `-check-format`
- `-check-architecture`

### How "it must not compile" is enforced

Go cannot hash a file at compile time, so enforcement is split between the compiler and `make build`.

Plain `go build ./...` covers only the compiler checks. `make build` covers both, and it is what CI runs.

Contract changes break the build until the code matches.

#### Contract changes

- **New operation:** `Handler` gains a method. `var _ api.Handler = (*service.Bus)(nil)` in `internal/service` fails to compile until it is implemented.
- **Changed payload or parameter:** Generated types and method signatures change. Every caller fails to compile until updated.
- **Removed operation:** The old method on `service.Bus` is now unreachable. The `deadcode` gate (`golang.org/x/tools/cmd/deadcode`, pinned) fails.
- **Contract edited without regenerating:** `contractgen -check-generated` regenerates in memory and fails on any difference, including the fingerprint.

#### Code changes

- **Hand edit in `generated/`:** `-check-generated` fails.
- **A route, `http.ServeMux` or `HandleFunc` outside `generated/`:** `-check-architecture` fails, apart from the page routes in `internal/web`.
- **A `json:` struct tag outside `generated/`:** `-check-architecture` fails. Wire types exist only in generated code.
- **Writing `text/event-stream` outside `generated/`:** `-check-architecture` fails.
- **Serving a contract other than the embedded bytes:** Integration test fails. `/asyncapi.yaml` must equal `api/asyncapi.yaml` and its SHA-256 must equal `ContractSHA256`.

## Stream identity

A stream ID is derived from a JSON Schema.

The same schema always gives the same ID, on every machine.

### Schema admission rules

A schema is rejected with a documented 400 response unless all of these hold:

- It is a JSON object (not a boolean schema) no larger than the configured size limit.
- `$schema` is exactly `https://json-schema.org/draft/2020-12/schema`.
- It validates against the official 2020-12 meta-schema, embedded unchanged. The bus never fetches anything over the network.
- Every `$ref` and `$dynamicRef` resolves inside the document (`#...` pointers, `$defs`, or embedded `$id` resources). Remote references are rejected.
- It compiles with the JSON Schema library.
- `format` is treated as an annotation, which is the 2020-12 default.

### ID derivation

The stream ID is:

```text
streamId = "sha256-" + lowercase_hex(SHA-256(JCS(schema)))
```

JCS (RFC 8785) removes differences in whitespace, key order and number formatting.

The hash covers the whole document, including `title` and `description`.

Two schemas that differ only in a description are two streams.

The ID is URL-safe and is validated against the pattern `^sha256-[0-9a-f]{64}$` wherever it appears in a path.

The complete schema document is hashed, not only its validation assertions. Changing `title`, `description`, examples or any other member creates a different stream ID.

`internal/streamid` owns canonicalisation, hashing and admission.

Its tests include the RFC 8785 test vectors.

## Authentication

Authentication is built into Pulsyflux. It is not a separate service, process, standalone application or independently deployed module. Its authentication and authorisation workflow is implemented by an internal subscriber with the reserved `Security` role, using the same subscription and publishing mechanisms as other subscribers.

### Unified credential authentication

All subscriber roles use the same bearer-token authentication path and the same in-memory credential store. Pulsyflux hashes the presented token, matches it to a stored credential hash, and reads the matched record's subscriber identity, role, stream scope and status. Role and scope are checked for every operation. A subscriber cannot choose its own role.

The four roles are:

- `Application`: external subscribers approved per Subscriber ID + Stream ID.
- `System`: internal Pulsyflux subscribers to a stream, created and managed by Pulsyflux.
- `Security`: reserved internal Pulsyflux subscriber for authentication and authorisation, created and managed by Pulsyflux.
- `Admin`: one global administrator credential, not scoped to any stream.

Credential records contain token hashes, not plaintext tokens. Application, System and Security credentials are scoped to their subscriber–stream relationship. The Admin credential is global.

The Security role is reserved and internal only. Pulsyflux creates its identity and credential during startup. It cannot be requested, approved, duplicated, revoked through subscriber workflows or assigned by an external client. The role is established from Pulsyflux's trusted internal startup provisioning; a client-supplied role name or identity is never sufficient proof.

### Admin credential and startup
There is no administrator username, password, account registration, separate browser session or administrator credential file.
At startup Pulsyflux reads `PULSYFLUX_ADMIN_TOKEN`, which must contain a cryptographically random secret with at least 256 bits of entropy. Pulsyflux hashes it and creates exactly one global Admin credential record in memory. The raw token is not retained, logged, traced, exposed by an API or placed in a URL.

The environment variable is mandatory. If it is missing or empty, Pulsyflux fails startup; it must never run without its single Admin record. The Admin record cannot be created, approved, duplicated or revoked through subscriber workflows. Changing the token requires restarting Pulsyflux.

The HTML administration page authenticates as an Admin subscriber by presenting this bearer token through the same hash lookup and role resolution as every other subscriber. Admin can administer every stream. Selecting a stream selects the target of an operation, not the scope of the Admin credential.

### Application subscriber identity

Each external Application subscriber supplies a stable Subscriber ID (a UUID generated by the subscriber installation and reused across subscriber restarts) and a human-readable Name. Subscriber IDs are normalised to lowercase. Names are required, trimmed, limited to 64 Unicode characters and rejected if they contain control characters. The Name is descriptive only, may be changed, and is maintained separately for each Subscriber ID + Stream ID relationship. Neither field is a credential.

### Application approval workflow

A subscriber requests access by attempting to subscribe to a stream; there is no separate registration workflow.

1. The subscriber attempts an SSE subscription and supplies its Subscriber ID and Name.
2. If an approved credential record exists for that exact Subscriber ID + Stream ID, the subscriber presents its bearer credential and Pulsyflux authenticates it.
3. If no approved relationship exists, Pulsyflux creates a pending request in memory and holds the SSE connection open for up to 10 minutes while awaiting approval. If approval is not received within that window, the pending request expires and is no longer eligible for approval; expiry is not an explicit denial.
4. The Admin UI lists pending requests for the selected stream. The administrator explicitly approves or rejects a request.
5. On approval, Pulsyflux generates a new Application credential, stores only its hash in memory, sends the plaintext credential once on the existing SSE connection through `pulsyflux.credential`, and allows message events to flow.
6. On rejection or timeout, Pulsyflux sends the same generic `subscription.closed` event with `{"code":"not_authorised"}`, flushes it and closes the connection. The subscriber cannot distinguish rejection from timeout.

An approved Subscriber ID + Stream ID relationship remains valid for the lifetime of the running process. Reconnection with its bearer credential does not require another approval. Approval for one stream never authorises another stream. A pending repeat request updates its Name without creating a duplicate or resetting the original deadline. Concurrent updates for a stream are serialised; the last processed update wins. A Name update for an approved relationship requires its valid bearer credential.

### Credential generation, storage and revocation

Pulsyflux generates Application, System and Security credentials using Go's `crypto/rand`: 32 random bytes (256 bits), encoded with `base64.RawURLEncoding`. It retains only the SHA-256 digest in memory and compares digests in constant time. Returning subscribers send credentials in the HTTP `Authorization: Bearer <credential>` header.

Credentials and hashes must never appear in URLs, query parameters, traces, ordinary logs, error messages or the administration interface. Credential delivery is one-time, uses cache-prevention headers and is excluded from SSE replay.

An authenticated Admin may revoke an approved Application relationship. Revocation immediately invalidates its credential and closes its active SSE connection; queued but undelivered messages are discarded. If a subscriber loses its credential, the administrator revokes the relationship and the subscriber requests approval again. An Application relationship may have at most one active SSE connection; a second simultaneous connection receives HTTP `429 Too Many Requests` and leaves the existing connection open.

System credential lifecycle is managed internally by Pulsyflux. Admin is global and cannot be revoked through the subscriber approval workflow.

### In-memory lifetime

All pending requests, credential hashes, role/scope records, the single Admin record, active connections and revocation state are in memory only. Pulsyflux creates, reads and writes no authentication-related files.

Process termination or restart clears all authentication state. Startup recreates exactly one Admin record from `PULSYFLUX_ADMIN_TOKEN`; Application subscribers must request approval again, and System subscribers are recreated internally.

### Security boundaries

The service is loopback-only and applies the Host, Origin, same-origin and Content-Security-Policy restrictions in Local-only rules. The operator must protect `PULSYFLUX_ADMIN_TOKEN` through the environment and service-manager configuration. Environment variables may be visible to sufficiently privileged local processes; this is an operating-system deployment boundary.

Authentication is programming-language independent and uses documented HTTP and SSE interfaces. A Go client library is optional.

## Intended contract surface

This is the starting point for the spec, not the final contract.

All paths are under `/v1`.

### API rate limiting

Every HTTP API endpoint is subject to standard HTTP request rate limiting. There are no exemptions for public, read-only or discovery endpoints: this includes schema retrieval, stream listing, health, request/response operations and the HTTP requests that establish SSE connections. Public endpoints remain unauthenticated where specified; rate limiting is an independent traffic-control measure, not an authentication or authorisation mechanism.

Use a standard token-bucket or sliding-window mechanism; do not build a bespoke rate-limiting subsystem. Before core credential verification establishes an authenticated principal, account the request by client IP address. Once valid credentials establish the principal through the existing shared credential model, account subsequent requests by that authenticated principal. A new subscriber's pending authorisation SSE connection remains accounted by client IP until a credential has been issued and presented on a later request; a client-supplied Subscriber ID is not an authenticated identity. The limiter observes identity established by the existing authentication process and must not verify credentials, make authorisation decisions or influence Security's decisions. Endpoint categories may have different configurable thresholds to reflect their cost and purpose; support normal burst handling. Reject requests exceeding their applicable limit with HTTP `429 Too Many Requests` and provide `Retry-After` where practical. If the service is ever placed behind a proxy, trust forwarded client-IP headers only from explicitly configured trusted proxies; otherwise use the direct peer address.

HTTP request rate limiting and SSE concurrent-connection limiting are independent controls. The HTTP limiter applies to the request that establishes an SSE connection; it does not count events sent over an already-established connection or replace the existing one-active-connection-per-Application-subscriber/stream rule. The latter continues to reject a second simultaneous connection with `429` while leaving the existing connection open.

The exact thresholds, endpoint categories and configuration defaults remain to be specified before implementation.

### Register schema

`PUT /v1/schemas`

Body is a JSON Schema.

Returns `streamId` and the canonical schema.

Idempotent: registering the same schema again returns the same ID.

### Get schema

`GET /v1/streams/{streamId}/schema`

Public, read-only schema discovery. It requires no subscriber authentication and returns the registered JSON Schema so any client can recompute and verify the stream ID. It grants no publish, subscribe or administration permissions. The endpoint remains subject to the universal API rate-limiting requirement.

### Publish

`POST /v1/streams/{streamId}/messages`

Body:

```json
{
  "data": {}
}
```

`data` must validate against the stream's schema.

Returns `{ streamId, runId, seq }` after the message is appended.

### Subscribe

`GET /v1/streams/{streamId}/events`

SSE.

Resume exclusively with the standard `Last-Event-ID` HTTP header. There is no query-parameter alternative.

Application subscribers identify themselves with their Subscriber ID and Name when requesting a new relationship. A pending relationship waits for administrator approval for up to 10 minutes. If approval is not received within that window, the pending request expires and is no longer eligible for approval; this is not an explicit denial. No message events are delivered before approval. On approval within the window, Pulsyflux creates an Application credential record and delivers the bearer credential once on the existing SSE connection. If an already-approved SSE connection later times out due to inactivity, that connection closure does not revoke or otherwise change the authorisation.

Subsequent subscriptions present the credential in the HTTP `Authorization: Bearer <credential>` header. Pulsyflux hashes the token, matches it against the in-memory credential store and uses the matched record's role and scope for authorisation.

The Admin UI presents the global Admin bearer credential through the same authentication mechanism. Admin authority is not stream-scoped; the stream in the request identifies the target of the operation. System and Security subscribers are internal Pulsyflux subscribers and use the same credential verification mechanism.

### List streams

`GET /v1/streams`

This is a public, read-only stream-discovery operation. It does not require subscriber authentication. It exposes registered stream metadata and aggregate operational counts only; it does not grant access to publish, subscribe, administer relationships or perform any other privileged operation. Its public-read behaviour is not a second authentication or authorisation model. The endpoint is subject to the universal API rate-limiting requirement.

Returns every registered stream with:

- Its schema `title`.
- Registration time.
- Last `seq`.
- Retained count.
- Live subscriber count.
- Accepted publish count.
- Rejected publish count.

### Stream trace

`GET /v1/streams/{streamId}/trace`

SSE of the complete trace for one stream, in trace order. The endpoint is publicly reachable as an API interface, but trace data is not anonymously accessible. An external client establishes the SSE connection and enters the ordinary external `Application` subscriber authorisation workflow, identifying its Subscriber ID and the requested Stream ID. Authorisation is granted or denied at the Stream ID scope: an Application subscriber approved for that stream can see all trace events for it, including activities involving other subscribers and publishers. Trace visibility is not restricted to the requesting subscriber's own activity, because stream-wide visibility supports debugging across the stream. The core holds the connection pending while the request is processed through the standard Security pub/sub authorisation workflow and emits no trace records unless access is approved. Once approved, the client receives the stream's retained historical trace as well as subsequent live events. If no `Last-Event-ID` is supplied, replay begins with the earliest retained trace event; if a cursor is supplied, the existing trace resume, replay and `trace.gap` rules apply. Authorisation does not limit history to events recorded after approval. The same credential verification, authorisation timeout, denial, revocation and connection rules that apply to an ordinary external subscription apply here; this endpoint introduces no separate authentication or authorisation path. The internal Tracing `System` subscriber also uses the standard Security workflow and receives no exemption.

Resume exclusively with `Last-Event-ID` (a `traceSeq`).

### Health

`GET /health`

Returns readiness and stream/subscriber counts.

### Authentication and administration operations

Authentication uses a single bearer-credential mechanism and in-memory credential store for all four roles: `Application`, `System`, `Security` and `Admin`. `Security` is reserved and internally provisioned by Pulsyflux; external subscribers cannot request or claim it. The contract must define subscriber identity fields, credential issuance and presentation, and role-based authorisation behaviour.

Administration operations must cover:
- Discovering available streams through the public, read-only `GET /v1/streams` operation, which requires no subscriber authentication and remains subject to API rate limiting.
- Retrieving pending Application subscribers and approved relationships for a selected stream.
- Approving or rejecting a pending Application subscriber–stream relationship.
- Revoking an approved Application subscriber–stream relationship.

The Admin UI authenticates using the single global Admin credential provisioned from `PULSYFLUX_ADMIN_TOKEN`. It uses the same bearer-token mechanism as every other subscriber; there is no separate administrator browser session. Every administration operation must require the matched credential record to have the `Admin` role.

The subscription approval workflow is part of the stream subscription process. It must not introduce a separate subscriber registration-request workflow.

### Message envelope

Envelope on the message SSE stream:

```json
{
  "streamId": "sha256-...",
  "runId": "...",
  "seq": 1,
  "publishedAt": "...",
  "data": {}
}
```

`runId` identifies the process run that assigned the message sequence.

The SSE `id` is the composite cursor `{runId}:{seq}`.

A gap is sent as a separate SSE event type, `stream.gap`, with:

```json
{
  "runId": "...",
  "fromSeq": 1,
  "toSeq": 5
}
```

The range is inclusive and identifies message sequence numbers that are no longer available to that subscriber.

Its SSE `id` uses the same run-scoped cursor format with `toSeq`.

Envelope field names are the same on every path.

A field is never renamed between publish and subscribe (for example `eventType` on one and `type` on the other).

## Broker semantics

### Ordering and identity

- The process creates one unique `runId` at startup.
- Every registered stream has its own monotonic `seq`, starting at 1 within that run.
- A `seq` is never reused during that process lifetime, including when retention evicts messages.
- The message identity is `(streamId, runId, seq)`.
- The first release has no stream-clear or stream-delete operation.
- Because stream and message storage are in memory, a process restart removes all streams. Registering the same schema in the new process starts a new stream history at `seq` 1 under the new `runId`.
- Delivery to one subscriber is in `seq` order.
- A publish returns only after the message is appended. No fire-and-forget work happens after the success response.
- `publishedAt` is assigned when the message is appended and is informational; `seq`, not the timestamp, defines stream order.

### Retention and replay

- Each stream keeps a bounded log, limited by message count and total bytes from configuration.
- A subscription with no resume cursor replays all messages currently retained for that stream, in `seq` order, then continues with live messages.
- A subscription with a valid cursor from the current `runId` replays every retained message after that cursor, then continues live.
- The broker validates the run-scoped cursor supplied in `Last-Event-ID`. If it names the current `runId` but its `seq` is greater than the latest sequence assigned in that run, the cursor is invalid for replay: replay all messages currently retained for the stream in `seq` order, then continue live. Do not treat the future sequence as a position to wait for, and do not emit a `stream.gap` for it; no retained message has been established as missing. This is broker-owned cursor validation and recovery behaviour, not a client-side validation responsibility.
- If `Last-Event-ID` is present but cannot be parsed as the required `{runId}:{seq}` cursor, treat it as invalid for replay: replay all messages currently retained for the stream in `seq` order, then continue live. Do not reject the SSE request solely because its resume cursor is malformed, and do not emit a `stream.gap`; a missing range has not been established. Cursor parsing and this fallback are broker-owned responsibilities.
- A cursor from an older `runId` is treated as invalid for replay: the server replays all messages currently retained for the stream in the current run, in `seq` order, then continues live.
- When messages between a valid current-run cursor and the oldest retained message were evicted, the subscriber gets one `stream.gap` event before the replay. A gap is never silent.
- For a valid current-run cursor with sequence `seq` and oldest retained sequence `oldestSeq`, emit exactly one inclusive `stream.gap` for `fromSeq = seq + 1` through `toSeq = oldestSeq - 1` only when `seq + 1 < oldestSeq`; then replay retained messages starting at `oldestSeq` in sequence order. When `seq + 1 == oldestSeq`, emit no gap and begin replay immediately. Never emit an empty or overlapping gap, or describe a retained message as missing. This boundary rule applies uniformly to all subscribers, including Security and Tracing; each subscriber's already-settled response to a gap remains specific to its correctness requirements.
- A message resume cursor is run-scoped (`{runId}:{seq}`).
- The server must never compare a cursor's `seq` against a different run's sequence history.
- If the cursor names an older `runId`, the server treats it as invalid for replay and falls back to replaying all messages currently retained in the current run before continuing live.
- The message resume cursor is supplied through the standard `Last-Event-ID` HTTP header only. No query-parameter alternative is supported.

### Subscribers and backpressure

- Publishers never block on subscribers.
- Each subscriber has a bounded data queue and a separate reserved slot for one terminal control event. The reserved slot does not increase the number of messages the subscriber may buffer.
- If a live publish cannot be added to a subscriber's full data queue, detach that subscriber from future fan-out.
- Drain messages already accepted into its queue in `seq` order, send one final `stream.gap` through the reserved control slot, then close the subscription.
- Other subscribers are unaffected.
- The overflow `stream.gap` has inclusive `runId`, `fromSeq` and `toSeq` fields identifying the contiguous range of messages not delivered because the subscription terminated.
- For live-delivery overflow, the range starts with the first message that could not be queued.
- The subscriber is detached atomically with overflow detection, so later publishes are not part of that subscription's gap.
- Replay obeys the same bounded data-queue limit as live delivery.
- If replay cannot fit, stop queuing replay, detach the subscriber from future fan-out, drain the messages already accepted into its queue, send a final `stream.gap` covering the remaining messages in that replay snapshot, then close the subscription.
- A terminal gap must not be dropped because the data queue is full.
- A subscriber that overflows receives its accepted queued messages before the terminal gap; it does not receive messages published after it is detached.
- The subscription is registered before the SSE response headers are flushed.
- The first frame is the comment `: subscribed`.
- Clients and tests wait for that comment, not for a sleep.
- Heartbeats are SSE comments (`: heartbeat`) at a configured interval.
- A subscription awaiting administrator approval is not permitted to receive message events. The approval waiting process must not allow messages to bypass authentication.
- An approved pending subscription continues on its existing SSE connection.

### Concurrency rules

These are the usual defects of in-memory brokers. Each one is forbidden:

- Subscription registration and replay-snapshot capture are atomic with respect to publishing under the stream lock. The broker registers the subscriber and captures the retained replay snapshot as one ordered handoff, so every publish is either included in that snapshot or queued for that subscriber as live delivery; no publish can fall between the two phases.
- The subscription owner delivers the captured replay snapshot in `seq` order before draining messages queued for live delivery. Live publishes that occur after registration are queued while replay is being delivered and cannot overtake replay messages. The handoff does not hold the stream lock while sending to the subscriber. Replay and live messages share the subscriber's existing bounded data-queue limit and overflow handling; this ordering guarantee does not create an unbounded buffer.

- Never send on a channel while holding a stream lock that a reader needs. Replaying into a bounded channel under the stream lock deadlocks the stream once the replay is larger than the channel.
- Only the subscription's owner closes its channel. Fan-out never sends to a channel that can be closed concurrently, because that panics.
- Remove subscribers by identity, never by a captured slice index. Indexes shift when earlier subscribers leave.
- Lifecycle hooks (first subscriber, last subscriber left) fire exactly once per transition. The first release has no stream-clear operation.
- Never iterate a shared map after releasing its lock.

### Errors

- Errors that happen before the SSE stream starts use the documented HTTP status. Never return 200 and then report the error as a frame inside the stream.
- Request bodies have a configured size limit. Over the limit returns 413.
- Every error is a generated, documented response type.
- A pending subscription rejected by the administrator or not approved within the allowed waiting period receives the same terminal SSE event: `subscription.closed` with `{"code":"not_authorised"}`; Pulsyflux flushes the event and closes the connection.
- Rejection and timeout must not be distinguishable by the subscriber. The actual outcome may be recorded internally.
- Because the SSE response has already started with HTTP `200 OK`, its status cannot be changed; errors detected before SSE starts use their documented HTTP responses.

## Tracing

The bus records what happens to publishes and subscriptions, in the exact order it happened.

Each stream has its own trace, written to a local JSON Lines file and served through the trace SSE operation.

### Trace events

`TraceEvent` is a contract schema in `api/asyncapi.yaml`, so its Go type is generated like every other wire type.

`runId` is generated once when the process starts and remains constant for that process lifetime. It is an opaque unique identifier, not an ordering key. It does not replace `traceSeq`.

### TraceEvent fields

- `traceSeq`: Per-stream order key. Starts at 1 for a new trace history and is strictly increasing. If no trace files remain when a stream is registered, the recorder starts a new trace history at 1; if the process restarts and retained trace files remain, it continues after the highest retained `traceSeq`. When a new history starts after the previous files are absent, an old SSE resume cursor cannot be distinguished from a cursor ahead of the new history; under the trace SSE resume rule, the server waits for future events with higher `traceSeq` values.
- `at`: UTC timestamp, RFC 3339 with nanoseconds. For display only; ordering always uses `traceSeq`.
- `kind`: One of the kinds below.
- `streamId`: The stream the event belongs to.
- `runId`: The process run that produced the trace event. Every trace event includes it, allowing retained trace files to distinguish events from different process runs.
- `subscriptionId`: Set on subscription and delivery events. Assigned by the bus per SSE connection (`sub-<n>`).
- `seq`: The message `seq`, when the event concerns a message; interpret it together with `runId`.
- `detail`: Kind-specific fields defined in the contract, such as a rejection reason, gap range or close reason.

### Trace event kinds

- `schema.registered`: A new schema is admitted (not on an idempotent re-registration).
- `publish.accepted`: A message is appended. `detail` has its size in bytes.
- `publish.rejected`: A publish to a known stream fails payload validation or the configured message-size limit. `detail` has the reason. A request for an unknown stream cannot be written to a per-stream trace and is represented only by the HTTP error response.
- `subscription.opened`: A subscriber connects. `detail` has the `Last-Event-ID` it resumed from, if any.
- `delivery`: A message frame has been written and flushed to a subscriber. This is the point where the publish "satisfies" that subscription.
- `subscription.gap`: A gap event is sent to a subscriber. `detail` has `runId`, `fromSeq` and `toSeq`.
- `subscription.overflow`: A subscriber's queue overflowed and it is being disconnected.
- `subscription.closed`: A subscription ends. `detail` has the reason (client left, overflow, shutdown).
- `trace.dropped`: The trace writer fell behind and dropped events. `detail` has the count.
- Authentication-related subscription decisions may also be recorded internally. They must not expose message payload data or reveal administrator decisions to unauthorised clients.

Message `data` is never copied into the trace in the first release.

Trace events may include message size and identifiers, but not message content.

### Ordering rules

`runId` is assigned once at process startup and is included in every message envelope and every `TraceEvent`.

`traceSeq` is assigned by the per-stream recorder while serializing calls to record events.

`publish.accepted` is recorded in the same broker critical section as the append.

`delivery` is recorded only after the SSE frame is flushed.

Therefore, a message's `publish.accepted` always has a lower `traceSeq` than each recorded `delivery` for that message.

A trace event that cannot be queued because the bounded writer queue is full is not assigned a `traceSeq`.

The recorder increments a pending-drop count.

Before it accepts a later event, it must enqueue one `trace.dropped` event containing that count. If it cannot enqueue the marker, the later event is also dropped and added to the pending count.

This preserves file order and makes loss visible without blocking publishers or deliveries.

`traceSeq` is contiguous across records that are actually written; the `detail.count` on `trace.dropped` reports omitted events.

Timestamps are never used for ordering.

Clock resolution on Windows is too coarse to separate events that happen close together.

Each stream's trace is ordered by its own `traceSeq` in the file, on the SSE operation and on the page.

The monitor page displays only the selected stream's trace, ordered by that stream's `traceSeq`; it does not merge events from multiple streams.

### Trace sinks

Each stream has its own trace file at:

`data/streams/{streamId}/trace.jsonl`

Tracing is enabled by default and cannot be disabled through runtime configuration in the first release.

The format is JSON Lines: one `TraceEvent` per line, UTF-8, in `traceSeq` order.

The file rotates at `trace.maxFileBytes`.

`trace.maxFiles` is the total number of retained files, including the active `trace.jsonl`.

Rotated files are named `trace.jsonl.1`, `trace.jsonl.2`, and so on.

When the limit is reached, the oldest file is deleted.

Trace files survive a process restart even though in-memory streams and messages do not.

When a schema is registered again, its recorder scans the retained trace files and continues `traceSeq` after the highest retained value.

When a recorder scans retained trace files on stream registration, it validates every retained file and every JSON Lines record needed to establish the trace history. If any retained file is unreadable or contains a malformed record, the recorder must not skip the file, truncate it, overwrite it or guess the next `traceSeq`. File tracing for that stream is disabled for the remainder of the process; no new events are appended for that stream. `/health` reports the affected stream and degraded trace status. Retained files are left untouched for operator inspection or repair. Other streams and the bus continue normally.

The files are written only to local disk.

One writer goroutine per stream owns its trace files. The recorder and writer are created when the stream is first registered and remain alive for the rest of the process lifetime, including periods when the stream has no subscribers. Subscription opening or closing does not stop or recreate the recorder.

On process shutdown, the recorder stops accepting new trace events, drains its queued events, emits any pending `trace.dropped` marker, flushes and closes the active file. On restart, recorders are recreated lazily when their streams are registered again; the recorder validates all retained trace files and records, then continues `traceSeq` after the highest retained value. If any retained file is unreadable or contains a malformed record, file tracing for that stream is disabled for the remainder of the process; no records are skipped, truncated, overwritten or appended, and no next sequence number is guessed. `/health` reports the affected stream as degraded, while the files remain untouched for operator inspection or repair.

Tracing is an internal System subscriber and publisher. Its publishing and subscription operations use the same standard credential verification and Security authorisation workflow as every other publisher and subscriber; its internal status grants no exemption, bypass or special broker privilege. Tracing's internal subscriber automatically reconnects after a disconnect or crash using its reserved, startup-provisioned System identity and credential. It resubscribes through the ordinary subscription mechanism and replays retained authorisation events through the normal SSE replay mechanism. If required events have been evicted, the ordinary `stream.gap` mechanism exposes the missing range. Tracing recovery does not block Security or the core, and Tracing receives no special broker privileges or unlimited queue.

When Tracing receives a `stream.gap` for its internal authorisation-event subscription, it records an explicit diagnostic in its own trace history identifying the affected source stream and missing sequence range. Tracing does not fabricate missing authorisation events or infer Security decisions. The gap places Tracing's health status in a degraded state.

A recorded authorisation-event gap places Tracing's health status in a degraded state. The degraded warning clears automatically when Tracing successfully reconnects to its internal authorisation-event subscription. The gap diagnostic remains in trace history; reconnection does not recover or fabricate missing events.

If Tracing disconnects again or detects another unrecoverable authorisation-event gap after recovery, its health status becomes degraded immediately when that condition is detected; no additional timeout is required.

Tracing retries its internal subscription using exponential backoff with a 2× multiplier and full jitter, avoiding synchronised reconnect attempts during a broker outage. After each unsuccessful attempt, the backoff cap doubles and is capped at the configured maximum; the actual retry delay is selected uniformly at random from zero up to that cap. A successful reconnection resets the backoff to its initial value.

The initial and maximum reconnect delays are deployment-configurable through `trace.reconnectInitialDelay` and `trace.reconnectMaxDelay`. The defaults are 1 second for `trace.reconnectInitialDelay` and 60 seconds for `trace.reconnectMaxDelay`; both remain deployment-configurable.

Producers hand events to it through a bounded queue, configured by `trace.queueSize` and defaulting to 10000 events.

Tracing never blocks a publish or a delivery.

When the queue is full, the recorder counts dropped events and emits a `trace.dropped` marker before the next event it successfully queues, as defined in Ordering rules.

A drop is never silent while the file sink remains operational.

On orderly shutdown, the writer emits any pending `trace.dropped` marker before closing the file.

The writer flushes to disk on a configurable interval, defaulting to 50ms.

Each flush drains all currently available events from the queue and writes them as a batch.

The interval is a runtime configuration value; it is not a per-request setting.

A file write or rotation error is logged once per stream and permanently disables that stream's file sink for the remainder of the process. Once disabled, that stream's recorder stops accepting new trace events and discards them immediately; they are not added to the writer queue and do not contribute to queue-overflow drop counts or `trace.dropped` markers. The sink is not retried or recreated during that process run; a process restart is required to restore file tracing for the stream.

`/health` reports the affected stream and degraded trace status.

The bus keeps running; it does not claim that tracing remains complete after the sink is disabled.

### Trace gaps

`trace.gap` is an SSE control event, not a persisted `TraceEvent`.

Its JSON data has `fromTraceSeq` and `toTraceSeq`.

Its payload schema and event name are still part of `api/asyncapi.yaml` and are generated and validated like the other public wire shapes.

`fromTraceSeq` and `toTraceSeq` are inclusive missing sequence bounds.

The trace SSE operation (`GET /v1/streams/{streamId}/trace`) tails the active and rotated files directly; there is no separate in-memory trace log.

Its resume cursor is `traceSeq`, supplied exclusively through the `Last-Event-ID` HTTP header.

No query-parameter alternative is supported.

If `Last-Event-ID` is present but cannot be parsed as a valid trace sequence cursor, the server treats it as invalid and replays the currently retained trace events from the earliest available event. It does not reject the trace SSE request or emit `trace.gap`, because a missing trace sequence range has not been established. Cursor parsing and this fallback are server-owned responsibilities.

The server locates the first retained event after that cursor. If the cursor is higher than the newest retained `traceSeq`, it treats the cursor as a valid resume position and waits for future events with higher `traceSeq` values; it does not reject the cursor or emit a `trace.gap` because no missing range has been established.

If rotation has removed events after the cursor, it first sends a `trace.gap` SSE event with `{ fromTraceSeq, toTraceSeq }`, where the range identifies the missing trace sequence numbers, then continues with retained events.

The SSE `id` on each persisted trace event is its `traceSeq`.

The `trace.gap` event uses `toTraceSeq` as its SSE `id`, so a reconnect after the gap does not report the same missing range again.

The trace SSE operation continues to use `traceSeq` as its SSE `id`, because that cursor remains continuous across process restarts.

Each persisted `TraceEvent` carries `runId`, so consumers can identify which process run produced it.

The trace is not itself a bus stream. This avoids recursion where tracing a delivery creates another delivery.

### Package

`internal/trace` owns the per-stream counter and the file writer.

The broker and the service receive a `*trace.Recorder` through their constructors.

It is a concrete type with no interface, because there is only one implementation.

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
- Pause closes the selected stream's trace SSE connection and retains the most recently received SSE event ID as the resume cursor. For persisted trace events this is their `traceSeq`; for a `trace.gap` control event it is the event's `toTraceSeq` ID.
- Resume opens a new trace SSE connection for the selected stream using the retained cursor in the standard `Last-Event-ID` HTTP header.
- Events still present in retained trace files after the cursor are replayed.
- Events removed by rotation after the cursor are reported through `trace.gap`, after which replay continues with retained events.
- Jump to `traceSeq`.

The page must not imply that events removed by rotation can be recovered.

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
internal/trace/                    per-stream trace counter and file writer
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
- Grants that account write access only to Pulsyflux's required configuration, state and trace directories.
- Supports install, upgrade and uninstall.

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

The former “Internal subscribers and the Security role” discussion has been reconciled with the authoritative settled decisions below. Internal subscriber identity and provisioning, the existing Security authorisation request/decision contracts and validation, delivery and failure behaviour, timeout handling, tombstone retention, and recovery are settled; they are not open design topics.

The universal HTTP rate-limiting requirement and its standard mechanism are settled: use identity/IP accounting, configurable endpoint-category limits, burst handling, standard `429` responses and trusted-proxy-only forwarded IP handling. Exact thresholds, endpoint categories and configuration defaults remain to be specified before implementation. HTTP request rate limiting is independent of the already-settled SSE concurrent-connection limit. The trace-retention scope is settled: each stream has one shared retained trace history visible to every authorised subscriber; subscriber-specific trace retention is not used. The trace-cursor-below-retained-history behaviour described in Settled decisions was explicitly agreed and is no longer an open topic.

Any newly identified decision that changes scope, externally visible behaviour or an acceptance criterion must be discussed and explicitly agreed before it is added to **Settled decisions**. Do not infer or silently settle decisions.

## Next-session handoff

**Resume from the latest `main` branch of this file.** Before proposing anything, read this handoff, the complete **Settled decisions** section, and the **Open decisions** section. Continue architecture discussion only: do not implement, scaffold, or create code. Discuss exactly one architectural decision at a time. For each proposal, explain both subsystem-specific behaviour and macro-architectural implications. A proposal explicitly marked NOT settled must remain unsettled until the user agrees. After each agreement, update this file directly on `main`, preserve all prior decisions, and verify the committed content by fetching it again.

**Last completed discussions:** The user agreed that Pause closes the selected stream's trace SSE connection and retains the most recently received SSE event ID as the resume cursor. Resume reconnects with that cursor in `Last-Event-ID`, relying on existing replay and `trace.gap` behaviour after rotation. The cursor includes the `toTraceSeq` ID when the last received event is a `trace.gap`. The user also agreed to the precise retention-eviction `stream.gap` boundary rule. For a valid current-run cursor at sequence `seq` and oldest retained sequence `oldestSeq`, emit exactly one inclusive gap from `seq + 1` through `oldestSeq - 1` only when `seq + 1 < oldestSeq`, then replay from `oldestSeq`. If `seq + 1 == oldestSeq`, replay immediately without a gap. Gaps must never be empty, overlap, or include retained messages. This rule applies uniformly to Security, Tracing and all other subscribers; their already-settled recovery responses remain distinct. The user has also agreed that all authorisation event JSON fields use camelCase: `correlationId`, `subscriberId` and `streamId`; that administrator approval/rejection instructions must enter Security's existing event-driven workflow through the dedicated internal authorisation stream; that the instruction event is named `authorization.administratorDecisionRequested`; and that the instruction payload contains exactly `correlationId`, `subscriberId`, `streamId` and `requestedDecision`, with `requestedDecision` exactly `Approved` or `Denied`. The correlation ID is the original core-generated request UUID; the operation is not repeated. The user has now also agreed to the administrator instruction authentication and structural validation contract: only the authenticated global Admin principal may submit; the core validates the exact four-field payload, UUID identifiers and decision enum before publication; invalid or unauthorised instructions are not published; Security remains the decision-maker and the core applies its validated decision. The user has also agreed to duplicate and stale administrator instruction handling: repeated instructions may produce repeated decisions, but the core's existing first-valid-decision rule prevents repeated or conflicting decisions from changing authorisation more than once; expired, unknown and already-acted-on correlations cannot create or revive requests or change permissions; no separate instruction deduplication registry, acknowledgement, retry or tombstone policy is added. The user clarified the 10-minute Admin-mediated Application approval window: an unapproved pending request expires after 10 minutes and is no longer eligible for approval; expiry is not an explicit denial. The ordinary 5-second Security response timeout does not expire this Admin-mediated request. If approval is accepted before the 10-minute window ends, later SSE inactivity/connection timeout does not reverse, revoke or otherwise change the established authorisation. These decisions are recorded in the authoritative sections below.

**Other recent decisions that must remain settled:**
- Security becomes unavailable immediately on subscription detachment, including queue overflow. It can resume only after replay proves uninterrupted continuity from its last successfully processed event position. An unrecoverable `stream.gap` permanently disables Security for the remainder of that process run; recovery requires restart. The core remains fail-closed under the previously agreed existing-connection policy.
- Internal subscribers use a distinct tagged internal-principal identity namespace, separate from Application Subscriber ID UUIDs. Reserved identities such as `security` and `tracing` are established only through trusted startup provisioning and credential records; this does not create a separate authentication or messaging path or grant blanket privileges.
- Tracing is observational. Its per-stream recorder lifecycle, shutdown draining, reconnect/backoff, gap diagnostics, sink write/rotation failure behaviour, and startup retained-file integrity validation are settled. A failed or corrupt stream sink is disabled for the rest of the process; no unsafe retry, queue accumulation, truncation, overwrite, or guessed sequence is permitted. Other streams, the bus and Security continue unaffected.
- Security's event-driven authorisation workflow, validation, deadlines, fail-closed handling, bounded workflow-owned tombstones, and recovery defaults are settled as recorded below. Preserve the distinction between Security's strict continuity requirement and Tracing's automatic warning clear after successful reconnection while retaining its gap diagnostic.

**Next session's first task:** Continue the full-document consistency review from the latest `main` commit, one issue at a time. The trace-retention scope decision is now settled: each stream has one shared retained history for all authorised subscribers. Do not reopen that decision or the other completed trace decisions above. Identify the next unresolved consistency issue in the document and present exactly one architectural decision at a time. The user has settled that stream listing (`GET /v1/streams`) and schema retrieval (`GET /v1/streams/{streamId}/schema`) are public, read-only discovery operations requiring no subscriber authentication. Every HTTP API endpoint must use standard HTTP rate limiting. The agreed approach is token-bucket or sliding-window rate limiting; authenticated requests are accounted by the existing authenticated principal, public unauthenticated requests by client IP, endpoint categories may have different configurable thresholds with burst handling, excess requests receive `429 Too Many Requests` with `Retry-After` where practical, and forwarded client-IP headers are trusted only from explicitly configured proxies. HTTP request rate limiting is distinct from the existing SSE concurrent-connection limit: it applies to the HTTP request establishing an SSE connection, not events sent over an established connection. Exact thresholds, endpoint categories and configuration defaults remain open for a later decision. Preserve the single shared authentication and authorisation model; do not introduce another mechanism, privileged bypass or internal-role exemption. Read this handoff, the complete **Settled decisions** section and **Open decisions** before continuing. Architecture/design only: no implementation, scaffolding or code changes without explicit authorisation. Discuss exactly one architectural decision at a time and verify any steering changes after committing directly to `main`.

**Most recent completed review decision:** The user selected shared trace retention per stream (Option A). Each stream has one retained trace history, and all authorised subscribers to that stream use the same history, rotation/eviction boundaries and replay/gap calculations. There are no subscriber-specific trace retention windows. This is recorded in the Tracing section and Settled decisions below.

**Most recent completed trace-review decisions:** The user confirmed that `GET /v1/streams/{streamId}/trace` is a publicly reachable SSE interface, but trace data is not anonymous: external clients establish the SSE connection and use the ordinary external `Application` subscriber Security pub/sub authorisation workflow. The internal Tracing `System` subscriber also uses the standard Security workflow and receives no exemption. Authorisation is scoped to the requested Stream ID; an approved Application subscriber sees the complete trace for that stream, including activity involving other subscribers and publishers, for cross-subscriber debugging. Visibility includes retained historical trace events from before authorisation as well as subsequent live events. With no `Last-Event-ID`, replay begins at the earliest retained trace event; with a cursor, existing trace resume, replay and `trace.gap` rules apply. The connection remains pending and emits no trace records until Security approves access. These decisions are recorded in the Tracing and Stream monitor sections and are settled.

**Recent completed review decisions:** The user agreed that a current-run message cursor ahead of the latest assigned sequence is invalid and causes the broker to replay all retained messages; a malformed message cursor has the same fallback, with no `stream.gap`. Subscription registration and replay-snapshot capture are atomic under the stream lock; the subscription owner delivers replay in `seq` order before queued live messages, using the existing bounded subscriber queue and overflow handling without holding the stream lock during sends. A malformed trace cursor replays all retained trace events from the earliest available event without emitting `trace.gap`. Earlier settled monitor Pause/resume semantics, trace future-cursor behaviour and message retention-gap boundaries remain unchanged.

**Latest verified review commit before this handoff update:** `c624fa972fb8dd7639c8988a81105316ddbb9f23` (`Include retained trace history after authorisation`). Other recent commits: `84025c80e2a9c75e01304ec214afca3c98a891c3` (ordered broker replay-to-live handoff), `86d04187bed7efb5a1af1e5718e97898e3b4cb35` (malformed message resume cursor), and `f3521ea6c5160cf0915cc7e8565a000f5c973b6c` (future message resume cursor).

No implementation or scaffolding is authorised.
## Settled decisions

The following points are settled by this steering file and are not open decisions:

- First-release stream and message storage is in-memory and is lost on process exit.
- All authentication and authorisation state is held in memory only; no authentication files are created or read.
- Authentication uses one shared in-memory credential store and one bearer-token verification mechanism for all subscriber roles.
- Every HTTP API endpoint uses standard HTTP rate limiting, including public read-only endpoints, health and SSE connection-establishment requests. Use a standard token-bucket or sliding-window mechanism. Before core credential verification establishes an authenticated principal, account requests by client IP; once valid credentials establish the principal through the shared credential model, account subsequent requests by that principal. A new subscriber's pending authorisation SSE connection remains accounted by client IP until a credential is issued and presented on a later request; a client-supplied Subscriber ID is not an authenticated identity. The limiter observes identity established by existing authentication and must not verify credentials, make authorisation decisions or influence Security's decisions. Endpoint categories may have different configurable thresholds and normal burst handling. Excess requests receive `429 Too Many Requests`, with `Retry-After` where practical. Forwarded client-IP headers are trusted only from explicitly configured proxies. HTTP request rate limiting is separate from SSE concurrent-connection limiting and does not count events on an established SSE connection. Exact thresholds, endpoint categories and configuration defaults remain open until specified before implementation.
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
- Trace retention is shared per stream, not per subscriber. Each stream has one retained trace history governed by its recorder's `trace.maxFileBytes` and `trace.maxFiles` settings. Every authorised subscriber to that stream replays from and observes the same retained history and the same retention-eviction boundaries; subscriber identity or connection lifetime does not create a separate retention window.
- Each stream's recorder and writer are created when its schema is first registered and remain alive until process shutdown, independent of subscriber activity. Orderly shutdown stops accepting trace events, drains queued events, emits any pending `trace.dropped` marker, flushes and closes the active file. On restart, recorders are recreated lazily when streams are registered and resume `traceSeq` after the highest retained value.
- Tracing's internal System subscriber automatically reconnects after disconnect/crash using its reserved startup-provisioned identity and credential, through the ordinary subscription mechanism and normal SSE replay. It receives no special broker privileges or unlimited queue, and its recovery never blocks Security or the core.
- An unrecoverable `stream.gap` is recorded in trace history with the affected source stream and missing sequence range. Tracing never fabricates missing events or infers Security decisions. Its health becomes degraded immediately on disconnect or unrecoverable gap, and the warning clears automatically on successful reconnection; the gap diagnostic remains in trace history. Repeated failures mark it degraded immediately.
- If a stream's trace file sink encounters a write or rotation error, it is permanently disabled for the remainder of the process. The recorder immediately stops enqueueing new events for that stream and discards them; they do not accumulate in the writer queue or increment queue-overflow drop counts or produce `trace.dropped` markers. The sink is not retried or recreated in the same process; restart is required to restore file tracing. `/health` continues to report the affected stream and degraded trace status, while the bus and Security authorisation continue unaffected.
- On recorder startup/stream registration, every retained trace file and JSON Lines record is validated before continuing `traceSeq` after the highest retained value. If any retained file is unreadable or contains a malformed record, the recorder does not skip it, truncate it, overwrite it, append to it or guess the next sequence number. File tracing for that stream is permanently disabled for the remainder of the process; `/health` reports the affected stream and degraded trace status, and retained files are left untouched for operator inspection or repair. Other streams and the bus continue normally.
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
- The trace flush interval defaults to 50ms and is configurable.
- The trace writer queue defaults to 10000 events and is configurable.
- There is no public stream-clear or stream-delete operation in the first release.
- SSE replay uses the standard `Last-Event-ID` HTTP header only. No `?lastEventId=` query parameter or alternative query-based replay mechanism is supported.
- The stream-monitor activity sparkline is calculated in the browser from the change in each stream's accepted publish count between successive `GET /v1/streams` polls. Rejected publish counts remain separate and are not included in the sparkline.
- When the stream monitor filters by subscription, it shows events belonging to that subscription and retains corresponding `publish.accepted` context events when they share the same `runId` and message `seq` with a visible `delivery`. It hides unrelated publishes and deliveries from other subscriptions; filtering changes presentation only and preserves the trace records and `traceSeq` order.
- On the stream monitor, Pause closes the selected stream's trace SSE connection and retains the most recently received SSE event ID as the resume cursor. For persisted trace events this is their `traceSeq`; for a `trace.gap` control event it is the event's `toTraceSeq` ID. Resume reconnects to the selected stream using that cursor in the standard `Last-Event-ID` HTTP header. The server replays retained events after the cursor and reports any events removed by rotation through the existing `trace.gap` mechanism; no client-side event backlog or new replay mechanism is introduced.
- For a syntactically valid trace resume cursor older than the earliest retained `traceSeq`, the server emits exactly one `trace.gap` with inclusive bounds `cursor + 1` through `earliestRetained - 1`, then replays from `earliestRetained`. The gap event's SSE ID is `toTraceSeq`, preventing the same missing range from being reported again after reconnect. This does not change malformed-cursor fallback (replay all retained events without a gap) or future-cursor behaviour (wait for future events without a gap).