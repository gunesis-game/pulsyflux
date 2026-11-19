
# Project steering: pulsyflux

Read this file first. It owns the mission, decisions and working rules for the repo. When the repo is created, this file moves to the repo root as `steering.md` in the first scaffold commit.

This file is self-contained. It does not depend on any other repo or steering file.

### Working rules

- All work is performed directly on `main`. Never create or use another branch.
- Do not create GitHub Issues or pull requests.
- The current phase is steering/design only. Do not implement software or scaffold the repository until the user explicitly authorises implementation.
- Update this steering file as decisions are agreed; do not silently treat proposals as settled decisions. Every rule needed to build the project is written here.

## Mission

Build one general-purpose message bus in Go: publishers, subscribers and a broker queue that serves them. Each stream is identified by the hash of a JSON Schema. The whole public surface is generated from a single AsyncAPI document. Applications that need messaging use this bus instead of building their own.

The bus runs only on the user's own machine. It is never exposed to the internet and never uses a cloud service.

## Current status

Nothing is implemented. The GitHub repo does not exist yet. This `steering.md` is the authoritative, self-contained source for the project's mission, requirements, decisions, working rules and acceptance criteria.

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

Authentication subscriber records are an explicit exception: every stream has its own local stream subscriber file containing that stream's pending and approved subscriber–stream records. These files survive process restarts. A subscriber authorised for multiple streams has a separate record in each stream's file.

Durable storage for streams and messages is out of scope unless separately specified.

### Docs UI

Served at `/`, with a "Try it" console generated from the contract.

### Tracing

Tracing is enabled by default.

Publish and subscription events for known streams are recorded per stream in a local JSON Lines file and exposed through the monitor page.

A file-sink failure is reported by `/health`; it does not stop the bus.

### Monitor UI

Served at `/streams`: stream list and live chronological trace per stream.

### Authentication administration UI

An HTML administration page is served locally by Pulsyflux.

It allows an administrator to authenticate and select individual streams to review pending subscriber approvals.

The authentication administration interface is part of the existing Pulsyflux service. It is not a separate application or service.

### Test libraries

Standard `testing` package only.

### Authentication

Authentication is built into the Pulsyflux service.

It is not a separate service, process or standalone application module.

Authentication is implemented as another internal subscriber to a stream, following the same general internal subscriber model used by tracing.

Authentication is responsible for:

- Managing administrator authentication.
- Managing subscriber approval.
- Maintaining per-stream pending subscriber records.
- Maintaining persistent subscriber–stream authorisation records.
- Establishing authenticated subscriber access.
- Managing subscriber credentials and credential security.
- Authorising stream subscriptions.
- Revoking subscriber access.

A subscriber installation identifies itself with a stable Subscriber ID and a human-readable Name. The Subscriber ID is a UUID generated by the subscriber and remains stable across restarts. The Name is descriptive only, may be changed by the subscriber at any time, and is stored separately for each subscriber–stream relationship. Neither field is a credential or proof of authorisation.

A subscriber requests access by attempting to subscribe to a stream.

If it has no previous authorisation for that stream, Pulsyflux holds the subscription pending.

The administrator approves or rejects the subscriber–stream access request.

An approved subscriber–stream authorisation is stored persistently.

The next time the same subscriber subscribes to the same stream, Pulsyflux recognises the existing authorisation and allows access after validating that relationship's bearer credential.

Authorisation is per subscriber, per stream. Approval for one stream does not automatically authorise access to another stream.

A subscriber that has no existing approval for a particular stream must obtain administrator approval for that subscriber–stream combination.

A subscription requiring administrator approval may establish an SSE connection while approval is pending. The connection remains open while the administrator considers the subscriber.

No message events are delivered until approval has been granted.

The approval wait is limited to 10 minutes. If approval is not obtained within that period, Pulsyflux closes the SSE connection. The client receives no information about whether the administrator approved or rejected the subscription. Pulsyflux records the decision internally.

The administration interface allows an authenticated administrator to select streams and review their pending subscribers. The interface displays each subscriber's stable Subscriber ID and its current Name for that stream.

Each stream has one dedicated subscriber-record file on local disk. It contains that stream's pending subscriber approval records and approved subscriber–stream authorisation records. Do not split pending and approved records into separate per-stream files, and do not use a shared cross-stream subscriber-record file.

Persistent approved subscriber–stream authorisations must also survive a process restart.

Pulsyflux is responsible for generating, validating, storing and revoking subscriber credentials. Subscriber applications must not need to implement their own cryptographic authentication modules.

Pulsyflux does not create subscriber applications or operate a subscriber-registration process. A subscriber requests access by attempting to subscribe.

Authentication must work through the documented HTTP and SSE interfaces, independently of the programming language used by a subscriber application.

A Go client library may be provided as an optional convenience. It must never be required to use Pulsyflux.

See Authentication for the complete requirements.

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
- Everything the service needs at runtime is embedded in the binary or read from local disk. That includes meta-schemas, the contract, the docs UI assets, authentication approval records and locally stored credentials.
- Configuration comes from a local file and command-line flags only.
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

Authentication is built into Pulsyflux. It is not a separate service, process, standalone application or independently deployed module.

Authentication is an internal subscriber to a stream, following the same general internal subscriber model used by tracing.

It is responsible for subscriber approval, administrator authentication, bearer credential management, authorisation and revocation.

The purpose of this design is to make secure messaging straightforward for subscriber application developers. Subscriber applications should not have to implement their own authentication infrastructure or cryptographic security modules.

Pulsyflux owns the security responsibilities.

Subscriber applications exist independently of Pulsyflux. Pulsyflux does not create subscriber applications or introduce a subscriber-registration workflow.

A subscriber requests access by attempting to subscribe to a stream and supplying its stable Subscriber ID and descriptive Name. Pulsyflux authorises access using the bearer credential associated with the specific subscriber–stream relationship.

Pulsyflux generates and manages the random bearer credential for an approved subscriber–stream relationship. After approval, Pulsyflux delivers the credential on the existing pending SSE connection using the dedicated `pulsyflux.credential` event, with a JSON `credential` field. Returning subscriptions present the credential only in the HTTP `Authorization: Bearer <credential>` header; credentials must not be placed in URLs or query parameters.

### Authentication architecture

Authentication is part of the existing Pulsyflux service and uses the existing stream and subscriber architecture.

It is not a separate authentication server.

The authentication component does not introduce a separate messaging architecture or an independent subscription mechanism.

It participates in the existing subscriber model.

The authentication component maintains the information needed to determine whether a subscriber may receive messages from a stream.

Tracing and authentication are both internal subscribers, but they have different responsibilities:

- Tracing records events about stream activity.
- Authentication manages subscriber approval and access.

### Subscriber identity

A subscriber installation must be distinguishable from other subscribers when requesting access to a stream.

**Settled decision:** Each subscriber supplies:

- **Subscriber ID:** a stable UUID generated by the subscriber installation. It identifies that installation across restarts and is the stable identifier used to associate the subscriber with its stream authorisations.
- **Name:** a human-readable display name supplied by the subscriber. It exists only to help the administrator recognise the subscriber. The subscriber may change it at any time.

The Name is stored per subscriber–stream relationship. The same Subscriber ID may therefore have different Names for different streams. On a subsequent request, the latest supplied Name for that relationship becomes the name shown in the administration interface.

Name changes are handled as follows:

- For an approved subscriber–stream relationship, Pulsyflux validates the relationship's bearer credential before accepting a Name update. A request with an invalid or missing credential must not change the stored Name.
- For a pending subscriber–stream relationship, a repeat request from the same Subscriber ID updates the existing pending record's Name. It does not create a duplicate pending record or reset the original 10-minute approval deadline.
- Updates to a stream's subscriber records are serialised. When requests for the same subscriber are concurrent, the last request processed while holding that stream's record lock determines the stored Name.
- A Name update affects only the requested subscriber–stream relationship. It does not change the Subscriber ID, credentials or Names stored for other streams.

The Subscriber ID and Name are identifiers and descriptive metadata only. Neither is secret, neither proves identity, and neither grants access. Authorisation is established by the credential associated with the specific subscriber–stream relationship.

The Subscriber ID must allow Pulsyflux to:

- Recognise the same subscriber installation across subsequent connections.
- Associate a subscriber with its previously approved stream authorisations.
- Distinguish subscriber installations from one another.
- Display a recognisable, subscriber-supplied Name to the administrator for each stream.

The exact validation rules for Subscriber IDs and Names, including length and character limits, remain to be specified.

### Subscriber approval

A subscriber requests access to a stream by attempting to subscribe to it and supplying its Subscriber ID and Name.

There is no separate registration-request workflow or registration-request endpoint.

A pending subscriber is represented by its attempted subscription to a particular stream, identified by the combination of Subscriber ID and Stream ID.

Authorisation is per subscriber, per stream.

Approval for one stream does not automatically authorise access to another stream.

The workflow is:

1. A subscriber attempts to subscribe to a stream and supplies its Subscriber ID and Name.
2. The SSE stream is opened.
3. Pulsyflux checks whether the subscriber already has approved access to that particular stream and, if so, validates its stream-specific bearer credential.
4. If no approved relationship exists, the SSE connection remains open while approval is pending, for a maximum of 10 minutes.
5. The pending subscriber is recorded in the pending approval file for that stream. The administrator can see its Subscriber ID and Name.
6. The administrator opens the authentication administration page and authenticates.
7. The administrator selects a stream.
8. Pulsyflux displays the pending subscribers for that stream.
9. The administrator explicitly approves or rejects the pending subscriber.
10. If approved, Pulsyflux creates a random bearer credential for that subscriber–stream relationship, stores the approved authorisation persistently, and allows the existing subscription to proceed. Normal SSE message events can then flow.
11. If the administrator rejects the request, Pulsyflux sends the terminal SSE event `subscription.closed` with the generic data payload `{"code":"not_authorised"}`, flushes it, and closes the SSE connection. The same event and payload are used when the approval waiting period expires; the subscriber cannot distinguish rejection from timeout.

A pending subscription does not grant access to stream messages.

Message events must never flow before approval.

There is no separate public registration-request resource that a subscriber creates and subsequently polls.

### SSE approval waiting period

The SSE connection remains open while the administrator considers the pending subscription.

The maximum waiting period is 10 minutes.

The connection is not closed immediately simply because administrator approval is pending.

If approval is granted before the timeout:

- The existing SSE connection proceeds.
- Normal stream events can flow.
- The subscriber does not need to establish a second subscription merely because approval was pending.
- The approved subscriber–stream authorisation is stored persistently.

If the administrator rejects the request or the waiting period expires without approval:

- Pulsyflux sends a terminal SSE event named `subscription.closed` with the generic data payload `{"code":"not_authorised"}`.
- Pulsyflux flushes the terminal event and then closes the SSE connection.
- Rejection and timeout use exactly the same event name and payload. The subscriber cannot distinguish the two outcomes.
- Pulsyflux records the actual outcome internally, without exposing the administrator's decision to the subscriber.

The SSE response has already started with HTTP `200 OK`; Pulsyflux cannot change its status after sending the response headers. Therefore, no HTTP timeout status is sent for an approval timeout or rejection after SSE has started. Errors detected before the SSE response starts use their documented HTTP status and error response.

### Per-stream pending approval files

Pending subscriber approvals are stored on local disk.

Each stream has its own pending approval file.

The file belongs to that stream and contains the pending subscriber information needed by the administrator interface and authentication component.

The files are organised by stream ID.

The intended location is:

```text
data/streams/{streamId}/pending-subscribers.json
```

The format is JSON.

The authentication component owns reading, updating and writing these files.

Updates must be safe against partial writes and concurrent approval activity.

An approval or rejection must be reflected consistently in the corresponding stream's pending approval file.

Pending approval records are separate from the in-memory message and stream storage.

They survive a Pulsyflux process restart.

Approved subscriber–stream authorisations must also be stored persistently on local disk and survive a process restart.

The exact persistent authorisation record format and storage arrangement remain to be specified.

The exact pending approval record fields and file-update implementation must be specified before implementation.

### Administrator authentication and administration page

Pulsyflux provides an HTML administration page for subscriber approval.

The administrator accesses the page through the local Pulsyflux service.

The page allows the administrator to provide their credentials.

Successful administrator authentication grants access to the administration interface.

The administrator can then:

- View the available streams.
- Select an individual stream.
- View the pending subscribers for that stream.
- Approve a pending subscriber.
- Reject a pending subscriber.
- Repeat the process for other streams.

The administrator's authentication is not tied to a single stream.

An authenticated administrator can select any stream available to the administration interface.

The administration page is part of `internal/web`, following the same embedded static HTML, CSS and JavaScript approach used by the stream monitor.

It does not require a separate frontend application or build step.

Administrator credentials and administrator access must be protected from ordinary subscriber applications.

An ordinary subscriber application must not be able to approve its own subscription or perform administrator actions.

The exact administrator credential provisioning and authentication mechanism remains to be specified.

### Credential generation and management

The agreed credential model is a random bearer credential generated by Pulsyflux for each approved subscriber–stream relationship.

Pulsyflux is responsible for:

- Generating a cryptographically random credential.
- Delivering the credential to the approved subscriber.
- Validating the credential on subsequent subscriptions.
- Associating the credential with exactly one Subscriber ID and Stream ID.
- Storing credential material securely on the local machine.
- Revoking credentials and subscriber access.
- Supporting credential replacement or rotation where specified.

A credential for one subscriber–stream relationship must not authorise access to any other stream or subscriber relationship.

The credential is a secret: possession of it grants the bearer the access associated with that subscriber–stream relationship, subject to revocation. The Subscriber ID and Name are not substitutes for the credential.

Subscriber applications must not be required to generate key pairs, sign challenges, or implement their own cryptographic challenge-response infrastructure. Pulsyflux handles credential generation and verification internally.

After an administrator approves a pending subscriber–stream request, Pulsyflux delivers the newly generated bearer credential to that subscriber on the existing pending SSE connection, using a dedicated `pulsyflux.credential` event. The event data is a JSON object with a `credential` string field, for example:

```text
event: pulsyflux.credential
data: {"credential":"<random-secret>"}
```

The credential is delivered only after approval and before message events are allowed to flow. The subscriber must retain the credential and present it when reconnecting, using the HTTP `Authorization: Bearer <credential>` header. Credential generation, encoding, server-side hash storage and replacement invalidation are specified below. Secure subscriber-side storage expectations and the exact recovery and replacement-delivery workflow remain to be specified.

### File terminology and credential-file binding

Use the established file terminology precisely:

- **Subscribe files** are called subscribe files.
- The **stream file** contains the subscriber records for that stream.
- Do not invent or refer to a separate "subscriber file". There is no such file in this design.
- When discussing subscriber records, identify them as records contained in the stream file.

#### Protection against stolen stream files

**Settled decision — authentication protection:** Possession of a stream file containing subscriber records alone must not allow an attacker to authenticate as an approved subscriber for that stream.

- Each approved subscriber–stream relationship has its own bearer credential, generated independently using 32 bytes (256 bits) of cryptographically secure randomness.
- The stream file stores only the SHA-256 digest of that credential. It never stores the plaintext bearer credential or a reversible representation of it.
- A presented bearer credential is hashed and checked against the stored digest using a constant-time comparison. The stored digest itself is not accepted as a bearer credential.
- Because the credential is uniformly random with 256 bits of entropy, possession of its SHA-256 digest does not make recovering the bearer credential computationally feasible.
- No separate persistent encryption key is required to meet this authentication-protection requirement.

This decision addresses theft or copying of the stream file only. Detection or prevention of unauthorised modifications to the file is a separate tamper-protection concern and is not settled by this decision.

### Credential generation, encoding and storage

**Settled decisions:**

- Pulsyflux generates each credential using Go's cryptographically secure random source, `crypto/rand`.
- Each credential contains 32 randomly generated bytes, providing 256 bits of entropy.
- The credential is encoded as unpadded Base64 URL-safe text (`base64.RawURLEncoding`), producing a 43-character token.
- Pulsyflux stores only the SHA-256 digest of the credential, never the plaintext credential, in the persistent subscriber–stream authorisation record.
- Credential validation hashes the presented credential and compares the digest using a constant-time comparison.
- Since the credential is uniformly random and has 256 bits of entropy, a password-specific slow password-hashing algorithm is not required.
- Plaintext credentials and credential digests must not be written to traces, ordinary application logs or error messages. The administrator interface must not display credentials.
- When a credential is replaced, the previous credential is invalidated immediately when the replacement is issued; there is no overlap period.

The plaintext credential exists only as needed to deliver it to the approved subscriber. Pulsyflux must not be able to recover an issued credential from its persistent authorisation records.

Credential delivery has the following additional requirements:

- Pulsyflux must durably persist the approved subscriber–stream authorisation record, including the credential digest, before attempting to deliver the plaintext credential.
- The `pulsyflux.credential` event is a one-time delivery. It must not be included in SSE replay or otherwise replayed after the original delivery attempt.
- Pulsyflux must not automatically resend a credential when delivery fails or when receipt is uncertain. An authenticated administrator must initiate credential replacement; replacement follows the established approval and invalidation rules.
- Authentication responses and the pending SSE response must use cache-prevention headers so credentials are not stored by HTTP caches.
- The credential must never appear in a URL, query parameter, trace, ordinary log, error response or administrator UI.

The subscriber application is responsible for securely retaining its credential. File-based storage is under consideration; the storage policy and exact subscriber-side storage expectations remain to be specified.

### Credential recovery and replacement

**Settled decisions:**

- Credential replacement can be initiated in either of two ways:
  - A subscriber can request recovery for its existing Subscriber ID–Stream ID relationship. An authenticated administrator must approve the request before a replacement credential is issued.
  - An authenticated administrator can initiate credential replacement directly, without a preceding subscriber recovery request.
- Knowing or presenting a Subscriber ID and Stream ID is not sufficient authority to replace a credential. A subscriber-initiated recovery request must be approved by an authenticated administrator.
- After the replacement is authorised, Pulsyflux generates a new credential using the established credential-generation rules and delivers it through the pending SSE connection using the dedicated `pulsyflux.credential` event.
- The recovery SSE connection may remain open for up to 10 minutes while the recovery request awaits administrator action or completion. If the recovery is not authorised and completed within that period, Pulsyflux closes the connection without disclosing the administrator's decision.
- If recovery times out before a replacement is issued, the existing credential remains valid and existing authenticated connections are not affected.
- When a replacement credential is issued, the previous credential is invalidated immediately, with no overlap. All existing SSE connections authenticated with the replaced credential are closed immediately.
- The subscriber must retain the replacement credential securely and use it for subsequent connections.

### Subsequent subscriptions

A subscriber that has already been approved should not need to repeat the complete administrator approval process every time it subscribes to the same stream.

Pulsyflux stores approved authorisation records persistently, scoped to the Subscriber ID and Stream ID.

The system must distinguish between:

- A subscriber attempting to access a stream for the first time.
- A subscriber that has already been approved for that stream and presents its valid bearer credential.
- A subscriber attempting to access a different stream for which it has not been approved.
- A subscriber whose access has been revoked.
- A subscriber presenting an invalid or expired credential.

Previously approved subscribers must be able to authenticate subsequent subscriptions using the bearer credential provisioned for that exact subscriber–stream relationship.

A subscriber's approval for one stream must not automatically authorise it for any other stream.

Approval and authentication are separate concerns:

- Approval determines whether a subscriber is permitted to access a stream.
- Authentication establishes that a subsequent connection possesses the credential associated with the approved Subscriber ID–Stream ID relationship.
- Each approved Subscriber ID–Stream ID relationship is limited to one active SSE connection at a time. A second simultaneous connection attempt using that relationship's credential receives HTTP `429 Too Many Requests`; the existing connection remains open and unaffected. Once the active connection closes, the credential may be used to establish a new connection.

The exact returning-subscription HTTP exchange and failure responses remain to be specified.

### Authentication and authorisation scope

Authentication and authorisation are stream-scoped.

Successful approval for a stream establishes that the identified subscriber is permitted to subscribe to that stream.

A separate roles-and-permissions system is not currently intended.

Each subscriber has independently managed access for each stream it uses.

Revoking one subscriber's access to a stream must not affect other subscribers' access to that stream.

Approval for one stream does not automatically grant access to another stream.

Previously approved subscriber–stream authorisations are persistent and survive process restarts.

### Publisher ownership

The publisher remains the owner of its stream as a working ownership concept.

Ownership does not currently confer exclusive control over subscriber admission or revocation.

Subscriber admission and credential revocation are controlled centrally by the Pulsyflux authentication component.

The precise remaining implications of publisher ownership are addressed under Open decisions.

### Programming-language independence

Authentication must be usable by applications written in any programming language capable of making HTTP requests and consuming SSE.

The public authentication mechanism must be documented and language-independent.

A subscriber application must not need to be written in Go.

A Go client library may be supplied as an optional convenience, but it must not be required.

The following must remain possible without an SDK:

- Subscribing to a stream.
- Establishing a pending SSE subscription.
- Obtaining administrator approval through the Pulsyflux administration process.
- Authenticating subsequent subscriptions.
- Publishing messages.
- Resuming subscriptions using the `Last-Event-ID` HTTP header.
- Handling authentication failures and credential revocation.

The HTTP and SSE contract remains the authoritative interface.

### Revocation

An administrator must be able to revoke a subscriber's access.

Revocation must affect only the relevant subscriber's access to the relevant stream. Other subscribers and other subscriber–stream relationships must continue operating normally.

When access is revoked, Pulsyflux immediately invalidates the affected relationship's credential and terminates every active SSE connection authenticated with it. No further messages may be delivered; queued but undelivered messages are discarded. Pulsyflux sends `subscription.closed` with `{"code":"not_authorised"}`, flushes the event, and closes each affected connection.

The revoked relationship cannot reconnect using its old credential. It must go through administrator approval again before a new credential is issued.

### Security boundaries

Pulsyflux must distinguish subscriber installations and operating-system users.

It must not assume that every local process is trusted merely because the service listens on loopback.

The implementation must consider:

- Subscriber applications running under different operating-system users.
- Unauthorised processes attempting to impersonate approved subscribers.
- Credential theft and reuse.
- Replay of authentication requests.
- Local credential disclosure.
- Credential revocation.
- Bearer credential rotation.
- Compromised subscriber credentials.
- Unauthorised access to per-stream pending approval files.
- Unauthorised administrator access.
- Attempts to approve subscriptions without administrator authentication.
- Attempts to obtain message events before approval.

## Intended contract surface

This is the starting point for the spec, not the final contract.

All paths are under `/v1`.

### Register schema

`PUT /v1/schemas`

Body is a JSON Schema.

Returns `streamId` and the canonical schema.

Idempotent: registering the same schema again returns the same ID.

### Get schema

`GET /v1/streams/{streamId}/schema`

Returns the registered schema so any client can recompute and check the hash.

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

Resume exclusively with the standard `Last-Event-ID` HTTP header.

There is no `lastEventId` query parameter or alternative query-parameter resume mechanism.

A subscription requiring administrator approval remains connected while approval is pending, for a maximum of 10 minutes.

Message events are not delivered before approval.

An approved subscription proceeds on the existing SSE connection.

If approval is not obtained within the waiting period, Pulsyflux closes the connection and uses the standard timeout behaviour without revealing the administrator's decision.

Subscribers are identified and authorised independently for each stream.

Previously approved subscriber–stream combinations are automatically authorised on subsequent subscriptions.

The exact authentication request information and subsequent-subscription authentication mechanism remain to be specified.

### List streams

`GET /v1/streams`

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

SSE of trace events for one stream, in trace order.

Resume exclusively with `Last-Event-ID` (a `traceSeq`).

### Health

`GET /health`

Returns readiness and stream/subscriber counts.

### Authentication and administration operations

Authentication-related public operations must be defined in `api/asyncapi.yaml`.

They must cover the documented subscriber authentication and subscription approval behaviour.

Administrator operations must cover:

- Administrator authentication.
- Access to the administration interface.
- Retrieving pending subscribers for a selected stream.
- Approving a pending subscriber.
- Rejecting a pending subscriber.
- Revoking previously approved subscriber access.

The subscription approval workflow is part of the stream subscription process. It must not introduce a separate subscriber registration-request workflow.

The administrator interface may use private local administration operations, provided they are documented and protected appropriately.

The exact public authentication paths, request payloads, responses and credential format must be defined in the AsyncAPI contract before implementation.

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
- A cursor from an older `runId` is treated as invalid for replay: the server replays all messages currently retained for the stream in the current run, in `seq` order, then continues live.
- When messages between a valid current-run cursor and the oldest retained message were evicted, the subscriber gets one `stream.gap` event before the replay. A gap is never silent.
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

- `traceSeq`: Per-stream order key. Starts at 1 for a new trace history and is strictly increasing. If the process restarts and the stream's trace files remain, the recorder continues after the highest retained `traceSeq`.
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

The monitor page merges multiple streams client-side using timestamps for display order, but ordering within a stream always uses `traceSeq`.

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

The files are written only to local disk.

One writer goroutine per stream owns its trace files.

Producers hand events to it through a bounded queue, configured by `trace.queueSize` and defaulting to 10000 events.

Tracing never blocks a publish or a delivery.

When the queue is full, the recorder counts dropped events and emits a `trace.dropped` marker before the next event it successfully queues, as defined in Ordering rules.

A drop is never silent while the file sink remains operational.

On orderly shutdown, the writer emits any pending `trace.dropped` marker before closing the file.

The writer flushes to disk on a configurable interval, defaulting to 50ms.

Each flush drains all currently available events from the queue and writes them as a batch.

The interval is a runtime configuration value; it is not a per-request setting.

A file write or rotation error is logged once per stream and permanently disables that stream's file sink for the remainder of the process.

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

The server locates the first retained event after that cursor.

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

`/streams` serves a page that lists every stream and shows each stream's trace live, in chronological order.

### Behaviour

#### Stream list

One card per stream, from `GET /v1/streams`.

Each card shows:

- The schema `title`.
- A shortened stream ID, with the full ID copyable.
- The counts.
- An activity sparkline.

The page refreshes the stream list by polling `GET /v1/streams` every 2 seconds. Newly registered streams then appear without a page reload.

The initial contract has no separate stream-list SSE operation.

#### Stream view

Selecting a card opens its trace from `GET /v1/streams/{streamId}/trace` as a vertical timeline, newest at the bottom, with auto-scroll that pauses when the user scrolls up.

#### Multi-stream view

The page opens one SSE connection per stream from `GET /v1/streams/{streamId}/trace` and merges them into a single timeline, with each event tagged by its stream colour.

#### Controls

- Filters by event kind and subscription.
- Pause and resume.
- Jump to `traceSeq`.
- On reconnect the page resumes using the standard `Last-Event-ID` HTTP header.
- Events still present in retained trace files are replayed.
- Events removed by rotation are reported through `trace.gap`.

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

- The page is embedded static HTML, CSS and JavaScript with no build step and no third-party libraries.
- It loads nothing remote and is covered by the same same-origin CSP header as the docs page.
- The page uses only contract operations: list streams, the trace operation and get schema.
- It has no private endpoints.
- `/` (docs) and `/streams` (monitor) link to each other.

## Authentication administration page

Pulsyflux provides an HTML administration page for administrator authentication and subscriber approval.

The page is served locally by the existing Pulsyflux service.

### Behaviour

#### Administrator authentication

The page provides an administrator credential form.

The administrator must authenticate before accessing pending subscriber information or performing approval actions.

An ordinary subscriber application must not be able to access administrator functionality.

#### Stream selection

After successful administrator authentication, the page displays the available streams.

The administrator selects an individual stream to review its pending subscribers.

#### Pending subscribers

For the selected stream, the page reads the corresponding pending approval information.

Each stream has its own pending approval file.

The page displays pending subscribers associated with that stream.

The administrator can approve or reject each pending subscriber.

Approval allows the associated pending SSE subscription to proceed, provided its waiting period has not expired.

Rejection closes the pending SSE subscription without disclosing the rejection to the subscriber.

Approval creates or updates the persistent authorisation record for that subscriber and that stream only.

#### Multiple streams

The administrator can move between streams and review their pending subscribers independently.

Approval or rejection for one stream must not accidentally change the approval state of another stream.

Approval for one stream does not authorise a subscriber to access another stream

### Visual design

The page should be clear and pleasant to use.

The selected stream must be obvious.

Pending subscribers should be easy to distinguish from previously handled entries.

Approval and rejection actions must be explicit and must not be triggered accidentally.

Light and dark themes follow `prefers-color-scheme`.

The layout works from laptop to wide screens.

### Rules

- The page is embedded static HTML, CSS and JavaScript with no build step and no third-party libraries.
- It loads nothing remote and is covered by the same same-origin CSP header as the docs page.
- Administrator credentials must not be exposed in page scripts, URLs or logs.
- State-changing administration operations must reject cross-origin requests.
- The page uses the existing Pulsyflux service and authentication component.
- It does not introduce a separate authentication application or service.
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
internal/auth/                     internal authentication subscriber, pending approvals, persistent subscriber–stream authorisations, administrator authentication, credentials, authorisation and revocation
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

Runs `verify`, then `test`, then builds `dist/pulsyflux`.

#### `make package`

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

Settle these in this `steering.md` before implementation.

### 1. Authentication and authorisation protocol

Settled:

- Authentication is built into Pulsyflux; there is no separate authentication service or process.
- Authentication is an internal subscriber, following the general internal subscriber model used by tracing.
- A subscriber requests access by attempting to subscribe to a stream.
- Each subscriber installation supplies a stable Subscriber ID, which is a UUID generated by the subscriber and reused across restarts.
- Each subscriber supplies a human-readable Name to help the administrator recognise it.
- The Name is descriptive only, is not used for identity or authorisation, and may be changed by the subscriber at any time.
- The Name is stored separately for each subscriber–stream relationship; the same Subscriber ID may use different Names for different streams.
- Authentication and authorisation are scoped to the subscriber–stream relationship. There is no application registration or application-wide credential.
- Approval for one stream does not grant access to another stream.
- Subscriber applications do not generate key pairs and do not need to implement authentication modules or their own cryptographic operations.
- Pulsyflux generates a random bearer credential for each approved subscriber–stream relationship.
- After approval, Pulsyflux delivers the credential on the existing pending SSE connection using the dedicated `pulsyflux.credential` event, with a JSON `credential` field.
- A bearer credential is specific to exactly one Subscriber ID and Stream ID.
- A bearer credential is valid only for the single Stream ID for which it was issued. Pulsyflux must reject it when presented for any other stream, including another stream to which the same Subscriber ID is approved. Credentials are never shared, reused or accepted across streams; each subscriber–stream relationship has its own independent credential.
- Credential validation must bind the presented credential to the requested Stream ID as well as the Subscriber ID. A valid credential for one relationship must never authenticate a request for another relationship.
- Each credential is generated from 32 bytes (256 bits) of cryptographically secure randomness using Go's `crypto/rand`.
- Credentials use unpadded Base64 URL-safe encoding (`base64.RawURLEncoding`).
- Persistent authorisation records store only the SHA-256 digest of each credential; plaintext credentials are not stored.
- Credential verification uses a constant-time digest comparison.
- Credential replacement immediately invalidates the previous credential; overlap is not allowed.
- Credentials and credential digests are excluded from traces, ordinary logs, error messages and the administrator interface.
- Returning subscriptions present the bearer credential only through the HTTP `Authorization: Bearer <credential>` header; credentials are never placed in URLs or query parameters.
- Pulsyflux manages credential generation, validation, storage and revocation.
- Pending and approved subscriber–stream records are maintained together in one dedicated local subscriber-record file per stream.
- An administrator authenticates through a local HTML administration page.
- The administrator selects streams and reviews pending subscribers for each stream, identified by Subscriber ID and Name.
- A pending SSE subscription remains open while approval is pending.
- The maximum approval waiting period is 10 minutes.
- Message events cannot flow before approval.
- An approved pending subscription proceeds on its existing SSE connection.
- An unsuccessful subscription is closed without revealing the administrator's decision.
- Pulsyflux records the decision internally.
- Subsequent subscriptions should not require the complete administrator approval process every time. A returning subscriber must present the bearer credential provisioned for that subscriber–stream relationship, exclusively in the HTTP `Authorization: Bearer <credential>` header.
- Approved subscriber–stream authorisations are stored persistently and survive a process restart.
- A Go client library is optional; authentication must work through documented HTTP and SSE interfaces.
- SSE replay uses the `Last-Event-ID` HTTP header only. There is no `lastEventId` query parameter.
- Subscriber IDs must be valid UUIDs supplied by the subscriber application and remain stable across application restarts.
- Subscriber IDs are normalised to lowercase before storage and comparison. A Subscriber ID is an identifier, not a secret or proof of authorisation.
- Subscriber Names are required and must not be empty or consist only of whitespace.
- Subscriber Names are limited to 64 Unicode characters.
- For approved relationships, a Name update requires successful validation of that relationship's bearer credential; invalid or missing credentials cannot change the Name.
- For pending relationships, a repeat request updates the existing Name without creating a duplicate pending record or resetting the original approval deadline.
- Concurrent subscriber-record updates are serialised per stream; the last request processed under that stream's record lock determines the stored Name.
- Name updates are scoped to one subscriber–stream relationship and do not alter identity, credentials or other streams' Names.
- Leading and trailing whitespace is removed from Subscriber Names before storage and display.
- Subscriber Names containing control characters are rejected. Other normal Unicode characters, spaces and punctuation are allowed.
- A Subscriber Name is descriptive only; changing it does not change the subscriber's identity or credentials.

Still to specify:

- The exact pending approval and persistent subscriber–stream authorisation record formats, including validation, atomic-write and corruption-recovery behaviour.

### 2. Publisher ownership

Define the remaining ownership responsibilities of a publisher when admission and revocation are controlled by the central authentication component.

### 3. Administrator identity and access

Specify:

- How the administrator's credentials are initially provisioned.
- How administrator credentials are stored securely.
- The administrator authentication mechanism.
- How administrator sessions are maintained and expired.
- How administrator approval actions are protected.
- Whether multiple administrators are supported.
- How administrator credentials can be changed or recovered.

The administrator interface must not allow an ordinary local subscriber application to approve subscriptions or revoke credentials.

### 4. Pending approval storage

Specify:

- The exact pending subscriber record fields.
- How pending records are added, updated and removed.
- How concurrent subscription attempts for the same subscriber and stream are handled.
- How approval files are updated atomically.
- How incomplete or corrupted files are handled.
- How pending records are reconciled with subscriptions after a process restart.
- How expired pending records are cleaned up.
- The exact format and storage arrangement for persistent approved subscriber–stream authorisation records.

The one-subscriber-record-file-per-stream storage model is settled: each file contains both pending and approved records for its stream. Each stream's subscriber-record file uses JSON (one JSON document per stream file, not JSON Lines). The JSON document has no format-version field and has this top-level structure:

```json
{
  "pending": [],
  "approved": []
}
```

The `pending` array contains pending subscriber approval records for that stream. Each pending record contains `subscriberId` (the subscriber's UUID), `name` (the subscriber-supplied display name for this stream) and `requestedAt` (the request time as an ISO 8601 UTC timestamp). The `approved` array contains approved subscriber–stream authorisation records for that stream. Each approved record contains `subscriberId` (the subscriber's UUID), `name` (the display name for this stream), `credentialHash` (the SHA-256 digest of the stream-specific bearer credential, encoded as 64-character lowercase hexadecimal; plaintext bearer credentials are never stored) and `approvedAt` (the approval time as an ISO 8601 UTC timestamp). Backwards compatibility with earlier file formats is not required; the implementation does not need to preserve compatibility when the format changes. The approved-record fields, validation rules, atomic-write procedure and other implementation details remain open.

### 5. Subsequent subscriptions

Specify the returning-subscriber authentication mechanism. It must authenticate the subscriber for the requested stream using the credential provisioned for that subscriber–stream relationship, without requiring the administrator approval process again.

The mechanism must not require subscriber applications to generate keys or implement their own cryptographic authentication module.

Authorisation must be checked for the specific subscriber–stream relationship. An approval for one stream must never be interpreted as approval for every stream.

### 6. Process model

Settled: The first-release installers configure and start Pulsyflux as a managed operating-system service.

Windows uses the Windows Service Control Manager and the unprivileged `LocalService` identity.

Linux uses a `systemd` unit and a dedicated unprivileged `pulsyflux` system account.

Foreground execution remains available for development and diagnostics, but is not the installed default.

## Settled decisions

The following points are settled by this steering file and are not open decisions:

- First-release stream and message storage is in-memory and is lost on process exit.
- Authentication pending approval records are stored on local disk, separately for each stream.
- Approved subscriber–stream authorisation records are persistent and survive process restarts.
- Streams use fan-out pub/sub (every subscriber receives every message) with replay, not competing-consumer queue semantics.
- The stream ID hashes the complete schema document, including annotations such as `title`, `description` and `examples`.
- The service is loopback-only and is not reachable from the LAN.
- Message content is never included in traces.
- The trace flush interval defaults to 50ms and is configurable.
- The trace writer queue defaults to 10000 events and is configurable.
- There is no public stream-clear or stream-delete operation in the first release.
- Authentication is built into Pulsyflux; there is no separate authentication service or process.
- Authentication is an internal subscriber, following the general internal subscriber model used by tracing.
- Subscriber installations exist independently of Pulsyflux and identify themselves with a stable Subscriber ID; their display Names are descriptive only.
- Subscribers request stream access by attempting to subscribe.
- There is no subscriber registration or subscriber registration-request workflow.
- Subscriber approval is required before message events can flow.
- Pending subscriptions can remain connected through SSE while awaiting administrator approval.
- The maximum approval waiting period is 10 minutes.
- An approved pending subscription proceeds on its existing SSE connection.
- Unsuccessful pending subscriptions are closed without disclosing the administrator's decision.
- Pulsyflux records approval decisions internally.
- Each stream has its own pending subscriber approval file on local disk.
- The administrator authenticates through a local HTML administration page.
- An authenticated administrator can select streams and review their pending subscribers independently.
- Subsequent subscriptions should not require the complete administrator approval process every time.
- Pulsyflux manages subscriber authentication and credential security.
- Subscriber applications do not generate key pairs or implement their own authentication or cryptographic operations.
- Pulsyflux generates a random bearer credential for each approved subscriber–stream relationship.
- Each credential contains 256 bits of cryptographically secure randomness and is encoded as unpadded Base64 URL-safe text.
- Pulsyflux stores only the credential's SHA-256 digest and validates it using a constant-time comparison.
- Replacing a credential immediately invalidates the previous credential.
- Credential recovery can be initiated by the subscriber (with administrator approval) or directly by an authenticated administrator.
- A Subscriber ID and Stream ID alone cannot authorise credential replacement.
- Credential recovery has a 10-minute maximum waiting period.
- Issuing a replacement credential immediately closes all existing SSE connections authenticated with the replaced credential.
- Authentication is programming-language independent and uses documented HTTP and SSE interfaces.
- A Go client library may be provided, but it is optional and never mandatory.
- Authentication and authorisation are scoped to a subscriber–stream relationship.
- Approval for one stream does not grant access to another stream.
- Previously approved subscriber–stream relationships are recognised on subsequent subscriptions using their stream-specific credentials.
- New subscriber–stream relationships require administrator approval.
- SSE replay uses the standard `Last-Event-ID` HTTP header only.
- No `?lastEventId=` query parameter or alternative query-based replay mechanism is supported.
