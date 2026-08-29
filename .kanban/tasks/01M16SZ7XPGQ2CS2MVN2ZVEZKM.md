---
assignees:
- claude-code
comments:
- actor: claude-code
  id: 01m16tg4k02ky96dmt6e5fq464
  text: |-
    Research complete. Findings:

    - `Sources/MCP/Base/Transports/HTTPClientTransport.swift` holds one `private var lastEventID: String?`. `processSSE(_:)` writes it for the POST response stream (call in `processResponse`) and for the standalone GET stream (call in `connectToEventStream`). `connectToEventStream` reads it for the `Last-Event-ID` header. This confirms the card.
    - Plan: add a private `SSEStreamKind` enum (`postResponse`, `standaloneGET`) and a `lastEventIDs: [SSEStreamKind: String]` dictionary. `processSSE(_:from:)` takes the kind and stores the id in its own slot. The GET header reads only `lastEventIDs[.standaloneGET]`. The POST path keeps its own slot, as the card orders.
    - Test seam: `Tests/MCPTests/HTTPClientTransportTests.swift` holds `MockURLProtocol` with an async handler. Deterministic order plan for the RED test: the POST response carries no session id header, so the standalone GET waits the full `sseInitializationTimeout` and every GET request is built AFTER `send()` returns. `send()` awaits `processSSE` to completion, so the shared field is set before any GET goes out. The POST SSE body carries `retry: 50` so the GET reconnects fast. The test records the `Last-Event-ID` header of each GET observed after `send()` returned, waits for two of them (the second is always built after the first was recorded), and asserts each is nil. Under the shared field the header carries the POST stream id, so the test fails before the fix on every schedule, not only when the race loses.
  timestamp: 2026-08-29T13:14:15.520300+00:00
- actor: claude-code
  id: 01m16tsj5c1c3e86qzntphpkff
  text: |-
    Implementation landed, TDD cycle complete.

    RED (attempt 1, dead end recorded for the next agent): the first test shape delivered NO session id, on the theory that the standalone GET would wait out `sseInitializationTimeout` and start after `send()` returned. It never started at all: without a session id the `initialSessionIDContinuation` never resumes, `await signalTask.value` in the task group never returns (the group awaits all children even after `cancelAll`, and `await task.value` on a non-throwing Task does not respond to cancellation), so the streaming task hangs forever. That is a separate latent defect — the sseInitializationTimeout does not actually release the GET when no session id ever arrives. Out of this card's scope.

    RED (attempt 2, correct failure): POST #1 returns text/plain with `Mcp-Session-Id`, which starts the GET loop. POST #2 returns an SSE body with `retry: 50` and `id: post-event-1`. `send()` awaits the full POST stream, so every GET built after `send()` #2 returns comes after the transport stored the id. The test waits for two GETs observed after that point (the second is always built after the first was recorded) and asserts each carries no `Last-Event-ID`. Before the fix: `lastEventID -> "post-event-1"` on both, on every run — no reliance on the race.

    GREEN: replaced the single `private var lastEventID: String?` with a private `SSEStreamKind` enum (`postResponse`, `standaloneGET`) and `private var lastEventIDs: [SSEStreamKind: String]`. `processSSE(_:from:)` stores each id under the kind of the stream it reads. The standalone GET header reads `lastEventIDs[.standaloneGET]` alone. The POST path keeps storing its own id in its own slot, as the card orders. The `retryInterval` field stays shared — the card does not name it.

    Full `swift test`: 552 tests in 40 suites, all pass, exit 0. No new warnings (the two `NoUseUnstructuredThrowingTask` warnings sit in `Tests/MCPTests/HTTPServerTransportTests.swift`, untouched by this change and pre-existing).

    Files changed:
    - Sources/MCP/Base/Transports/HTTPClientTransport.swift
    - Tests/MCPTests/HTTPClientTransportTests.swift (new test `testStandaloneGETDoesNotResumePOSTStreamEventID`)
  timestamp: 2026-08-29T13:19:24.332811+00:00
- actor: claude-code
  id: 01m16tt316bvq48rcckyekpxzt
  text: |-
    ### implement — changed
    - evidence: 2 files — Sources/MCP/Base/Transports/HTTPClientTransport.swift, Tests/MCPTests/HTTPClientTransportTests.swift; `swift test` 552 tests in 40 suites, all pass, exit 0
    - next: /review; the latent GET-never-starts hang is filed as ^s4qv5d6
  timestamp: 2026-08-29T13:19:41.606306+00:00
- actor: claude-code
  id: 01m16vjec9rnm0nkx96mb9ytyr
  text: |-
    ### review — findings
    - evidence: 17 findings from `review sha HEAD~1..HEAD` (commit 9aa6fd0). All findings are in Tests/MCPTests/HTTPServerTransportTests.swift at lines 212, 289, 327, 354, 437, 512, 581, 870, 1127. Two rule sets report them: `reuse/reuse` (8 items — the stream-drain pattern occurs 9 times; make one shared helper) and `swift/concurrency` (9 items — each unstructured `Task` must be stored and awaited, or replaced with structured concurrency).
    - next: correct the findings in the new dated `## Review Findings` checklist, then run `/review 01M16SZ7XPGQ2CS2MVN2ZVEZKM` again. The task stays in `review`.
  timestamp: 2026-08-29T13:32:59.657078+00:00
- actor: claude-code
  id: 01m16vk8626tamvr0zpttw5xvk
  text: |-
    ### finish iteration 1 — findings
    - implement: changed — 2 files (HTTPClientTransport.swift, HTTPClientTransportTests.swift); per-stream lastEventIDs with SSEStreamKind
    - test: green — swift test, 552 tests in 40 suites, 0 failures, 0 warnings on clean rebuild
    - commit: 9aa6fd0
    - review: findings — 17 open, all in Tests/MCPTests/HTTPServerTransportTests.swift (:212, :289, :327, :354, :437, :512, :581, :870, :1127); reuse of the stream-drain Task pattern, and unstructured Task values that are not stored and not awaited
  timestamp: 2026-08-29T13:33:26.082533+00:00
- actor: claude-code
  id: 01m16vs77bnad3zgpznkdm4vrz
  text: |-
    Research for the 17 findings in Tests/MCPTests/HTTPServerTransportTests.swift:

    - The drain pattern occurs at 9 sites. All drain an `AsyncThrowingStream<Data, Swift.Error>` from `HTTPResponse.stream`.
    - StatefulHTTPServerTransport closes a POST stream when its response routes (routeMessage: `continuation.finish()`), and closes ALL streams (request, standalone GET, incoming) in `disconnect()`/terminate. Thus an `await` on a drain task placed after `disconnect()` — or after the response is sent, for the two sites where the response closes the stream (initializeSession, testStatefulContextLifecycle) — cannot hang.
    - The file already holds a PRIVATE `drainSSEStream(_ response:maxChunks:timeout:)` helper that NO test calls (dead code). It also stores a throwing Task, cancels it, and never awaits it. Plan: replace it with a new shared `drainSSEStream(_ response: HTTPResponse) -> Task<Void, Never>?` that spawns the drain and returns the handle; every site stores the handle and awaits it at the end of the test.
    - Same cause elsewhere in the file, beyond the 9 listed lines: `defer { Task { await server.stop() } }` in testServerHandlerReadsHTTPContext (fire-and-forget unstructured Task) becomes a plain `await server.stop()` at the end; the throw path of `initializeSession` leaves `respondTask` running — it now cancels and awaits it before the throw.
    - All other Task values in the file are stored and awaited already.
  timestamp: 2026-08-29T13:36:41.707787+00:00
- actor: claude-code
  id: 01m16vyvkckmtxny8skp0bskfm
  text: |-
    Correction of the 17 review findings is complete. All work is in Tests/MCPTests/HTTPServerTransportTests.swift.

    What changed:
    - One shared helper `drainSSEStream(_ response: HTTPResponse) -> Task<Void, Never>?` now holds the drain pattern. It starts the drain task and returns the task handle.
    - The 9 drain sites now call the helper. Each site keeps the task handle and awaits it at the end of the test function, after the transport closes the stream.
    - Two sites await the handle after the response routes: `initializeSession` and `testStatefulContextLifecycle`. The transport closes a POST stream when its response routes. The other seven sites await the handle after `transport.disconnect()`. The transport closes all streams at disconnect. Thus no await can hang.
    - The old `drainSSEStream(_:maxChunks:timeout:)` helper had no callers. It also held a throwing task that no code awaited. The new helper replaces it.
    - The same cause was in two more places in the file. `testServerHandlerReadsHTTPContext` had `defer { Task { await server.stop() } }`; it now has a plain `await server.stop()` at the end. The throw path of `initializeSession` did not stop `respondTask`; it now cancels the task and awaits it before the throw.
    - All 17 checklist items in `## Review Findings (2026-08-29 08:27)` are now `- [x]`.

    Test result: `swift test` — 552 tests in 40 suites, all pass, exit 0. Build is clean, zero compiler warnings. Two "warning" lines in the test log are runtime logger output from the race-condition tests; they are not build warnings and they existed before this change.
  timestamp: 2026-08-29T13:39:46.412211+00:00
- actor: claude-code
  id: 01m16vyysgx7jata5dw68rpfqa
  text: |-
    ### implement — changed
    - evidence: 1 file — Tests/MCPTests/HTTPServerTransportTests.swift; `swift test` 552 tests in 40 suites, all pass, exit 0, zero compiler warnings; all 17 findings flipped to [x]
    - next: /review; the task stays in doing
  timestamp: 2026-08-29T13:39:49.680004+00:00
- actor: claude-code
  id: 01m16vzszjb9wt69yxxr0ptzaz
  text: 'Housekeeping note: the work tree held a stray, empty, untracked file with the literal name `Sources/MCP/*.swift` (zero bytes, created 08:23 by an earlier pipeline step, most likely a shell quoting accident). I removed it so the commit step does not add it to the repository. No other file outside Tests/MCPTests/HTTPServerTransportTests.swift changed.'
  timestamp: 2026-08-29T13:40:17.522707+00:00
position_column: doing
position_ordinal: '80'
title: HTTPClientTransport keeps one lastEventID for every SSE stream
---
## What

`Sources/MCP/Base/Transports/HTTPClientTransport.swift` holds ONE event id for
every SSE stream:

    90:    private var lastEventID: String?

`processSSE(_:)` (line 645) writes that one field at line 667:

    if let eventID = event.id, !eventID.isEmpty {
        self.lastEventID = eventID
    }

`processSSE` reads TWO different kinds of stream:

- line 361, the response stream of a `POST`
  (`let hadData = try await self.processSSE(stream)`);
- line 641, the standalone `GET` stream
  (`try await self.processSSE(stream)`).

The standalone `GET` then puts that one field on itself at line 602:

    if let lastEventID = lastEventID {
        request.addValue(lastEventID, forHTTPHeaderField: HTTPHeaderName.lastEventID)
    }

So the `Last-Event-ID` of the `GET` is the id of the last event that came in on
ANY stream, and usually that is a `POST` response stream.

## What it does to a server

`StatefulHTTPServerTransport.handleGet` reads the header, takes its resume path,
resolves the id to the stream that owns it, and binds the new `GET` to
`requestSSEContinuations[thatRequestID]` and not to `standaloneSSEContinuation`.
The session then holds no standalone stream, and `routeServerInitiatedMessage`
stores every `elicitation/create` and every notification "for replay" and
delivers none of them.

It is a race, and that is why it is intermittent. The `GET` goes out as soon as
the session id is read from the initialize response HEADERS, while that response
BODY is still parsing into `lastEventID`. Win the race and the field is empty and
the `GET` is clean. Lose it and the `GET` carries the id of a `POST` stream.

The MCP specification gives resumability a per-stream meaning: a client resumes
the stream it last read on THAT connection. One shared field cannot say that.

## How it was found

The FoundationModelsMultitool package measured it. About one full `swift test`
run in six timed out, because the standalone `GET` of its in-process HTTP
loopback resumed a `POST` stream. A trace of the loopback shows a clean run that
carries `id: _GET_stream_2` on the `GET`, and a red run that carries the event id
of a `POST` request.

That package repaired only the symptom, at the one seam it owns: its
`LoopbackHTTPServer` drops `Last-Event-ID` before it routes. That is correct for
a same-process loopback whose session never outlives a disconnect. It does
nothing for a real streamable-HTTP server, which is why this card stands.

## What to build

Make the id per-stream, and make the standalone `GET` read the id of the
standalone stream alone.

- Give `processSSE(_:)` a parameter that says which stream it reads, or divide
  the write so each caller stores into its own slot. The two call sites are line
  361 (POST response) and line 641 (standalone GET).
- The read at line 602 must take the id of the standalone stream alone. An id
  from a `POST` response stream must never reach that header.
- Keep the `POST` path storing its own id, because a `POST` response stream is
  resumable in its own right by the same specification.

## Acceptance Criteria

- [x] `HTTPClientTransport` holds no single shared `lastEventID`.
- [x] The standalone `GET` carries the id of the standalone stream, or no
      `Last-Event-ID` at all when that stream has read no event yet.
- [x] The event id of a `POST` response stream never reaches the `GET` header.
- [x] `swift test` passes.

## Tests

- [x] Add a test that drives a `POST` whose response stream carries an event id,
      then opens the standalone `GET`, and asserts that the `GET` carries no
      `Last-Event-ID` of that `POST` stream. `MockURLProtocol` in this
      repository's own test target is the seam.
- [x] Watch it fail before the fix. The race lets an unguarded test pass by luck,
      so drive the order of the two streams instead of waiting on it.

## Notes

- The defect is in `main` of the upstream repository, at `a0ae212`, which is also
  tag `0.12.1`. No branch of upstream corrects it: the one branch that touches
  `lastEventID` only changes a string literal to a constant, and its declaration
  at line 90 is identical.
- No issue and no pull request of upstream names `Last-Event-ID`, `lastEventID`
  or resumability. The defect is unreported.
- Upstream is close to dormant. `main` has not moved since 2026-04-29, and 66
  issues and 34 pull requests are open.
- The FoundationModelsMultitool package now depends on this fork. When this fix
  lands, that package takes it and removes the `Last-Event-ID` strip from its
  `Tests/Support/MCPTestServer/LoopbackHTTPServer.swift`. Its board holds the
  cards that record the measurement and the upstream report.

## Workflow
- Use `/tdd` — write the failing test first, then the fix. #transport #bug


## Review Findings (2026-08-29 08:27)

> Scope: `review sha HEAD~1..HEAD` — reviewed the diffs only — lines this change added or modified. 4 file(s) reviewed, 22 not reviewed.

> 21 file(s) not reviewed — excluded by an ignore rule:
> - `.kanban/ (from .reviewignore)` — 21 file(s)

> 1 file(s) not reviewed — no validator matched:
> - `.gitignore` — no validator matches this file

- [x] `Tests/MCPTests/HTTPServerTransportTests.swift:212` `reuse/reuse` — Stream-draining code `Task { do { for try await _ in stream {} } catch {} }` is repeated 9 times across this test file (lines 212–216, 289–293, 327–331, 354–358, 437–441, 512–516, 581–585, 870–874, 1127–1131). Extract to a shared test helper function to eliminate duplication. Create a helper function in the test file (e.g., `drainSSEStream(_ stream: URLSession.AsyncBytes)`) and call it instead of repeating the pattern 9 times. This keeps stream cleanup logic in one place and makes tests easier to maintain.
- [x] `Tests/MCPTests/HTTPServerTransportTests.swift:212` `swift/concurrency` — Unstructured Task spawned without storage or awaiting. The concurrency rule requires structured concurrency or long-lived Task handles that are cancelled in teardown. Store the Task handle in a collection and await it when no longer needed, or use structured concurrency (async let or withTaskGroup). For test cleanup, collect and await all drain tasks at the end of the test function.
- [x] `Tests/MCPTests/HTTPServerTransportTests.swift:289` `reuse/reuse` — Stream-draining code repeated: same pattern as line 212. Should use shared test helper instead. Extract to shared helper function.
- [x] `Tests/MCPTests/HTTPServerTransportTests.swift:289` `swift/concurrency` — Unstructured Task spawned without storage or awaiting. The concurrency rule requires structured concurrency or long-lived Task handles that are cancelled in teardown. Store the Task handle in a collection and await it when no longer needed, or use structured concurrency (async let or withTaskGroup). For test cleanup, collect and await all drain tasks at the end of the test function.
- [x] `Tests/MCPTests/HTTPServerTransportTests.swift:327` `swift/concurrency` — Unstructured Task spawned without storage or awaiting. The concurrency rule requires structured concurrency or long-lived Task handles that are cancelled in teardown. Store the Task handle in a collection and await it when no longer needed, or use structured concurrency (async let or withTaskGroup). For test cleanup, collect and await all drain tasks at the end of the test function.
- [x] `Tests/MCPTests/HTTPServerTransportTests.swift:354` `reuse/reuse` — Stream-draining code repeated: same pattern as line 212. Should use shared test helper instead. Extract to shared helper function.
- [x] `Tests/MCPTests/HTTPServerTransportTests.swift:354` `swift/concurrency` — Unstructured Task spawned without storage or awaiting. The concurrency rule requires structured concurrency or long-lived Task handles that are cancelled in teardown. Store the Task handle in a collection and await it when no longer needed, or use structured concurrency (async let or withTaskGroup). For test cleanup, collect and await all drain tasks at the end of the test function.
- [x] `Tests/MCPTests/HTTPServerTransportTests.swift:437` `reuse/reuse` — Stream-draining code repeated: same pattern as line 212. Should use shared test helper instead. Extract to shared helper function.
- [x] `Tests/MCPTests/HTTPServerTransportTests.swift:437` `swift/concurrency` — Unstructured Task spawned without storage or awaiting. The concurrency rule requires structured concurrency or long-lived Task handles that are cancelled in teardown. Store the Task handle in a collection and await it when no longer needed, or use structured concurrency (async let or withTaskGroup). For test cleanup, collect and await all drain tasks at the end of the test function.
- [x] `Tests/MCPTests/HTTPServerTransportTests.swift:512` `reuse/reuse` — Stream-draining code repeated: same pattern as line 212. Should use shared test helper instead. Extract to shared helper function.
- [x] `Tests/MCPTests/HTTPServerTransportTests.swift:512` `swift/concurrency` — Unstructured Task spawned without storage or awaiting. The concurrency rule requires structured concurrency or long-lived Task handles that are cancelled in teardown. Store the Task handle in a collection and await it when no longer needed, or use structured concurrency (async let or withTaskGroup). For test cleanup, collect and await all drain tasks at the end of the test function.
- [x] `Tests/MCPTests/HTTPServerTransportTests.swift:581` `reuse/reuse` — Stream-draining code repeated: same pattern as line 212. Should use shared test helper instead. Extract to shared helper function.
- [x] `Tests/MCPTests/HTTPServerTransportTests.swift:581` `swift/concurrency` — Unstructured Task spawned without storage or awaiting. The concurrency rule requires structured concurrency or long-lived Task handles that are cancelled in teardown. Store the Task handle in a collection and await it when no longer needed, or use structured concurrency (async let or withTaskGroup). For test cleanup, collect and await all drain tasks at the end of the test function.
- [x] `Tests/MCPTests/HTTPServerTransportTests.swift:870` `reuse/reuse` — Stream-draining code repeated: same pattern as line 212. Should use shared test helper instead. Extract to shared helper function.
- [x] `Tests/MCPTests/HTTPServerTransportTests.swift:870` `swift/concurrency` — Unstructured Task spawned without storage or awaiting. The concurrency rule requires structured concurrency or long-lived Task handles that are cancelled in teardown. Store the Task handle in a collection and await it when no longer needed, or use structured concurrency (async let or withTaskGroup). For test cleanup, collect and await all drain tasks at the end of the test function.
- [x] `Tests/MCPTests/HTTPServerTransportTests.swift:1127` `reuse/reuse` — Stream-draining code repeated: same pattern as line 212. Should use shared test helper instead. Extract to shared helper function.
- [x] `Tests/MCPTests/HTTPServerTransportTests.swift:1127` `swift/concurrency` — Unstructured Task spawned without storage or awaiting. The concurrency rule requires structured concurrency or long-lived Task handles that are cancelled in teardown. Store the Task handle in a collection and await it when no longer needed, or use structured concurrency (async let or withTaskGroup). For test cleanup, collect and await all drain tasks at the end of the test function.
