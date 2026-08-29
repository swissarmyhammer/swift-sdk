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
