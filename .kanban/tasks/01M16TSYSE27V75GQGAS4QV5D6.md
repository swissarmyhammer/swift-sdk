---
assignees:
- claude-code
comments:
- actor: claude-code
  id: 01m16wg29ne23e4ec6j9e4zemq
  text: |-
    Research and TDD progress:

    1. RED: Added the test `testStandaloneGETStartsWhenNoSessionIDArrives` in Tests/MCPTests/HTTPClientTransportTests.swift. The mock POST response has no `Mcp-Session-Id` header, and `sseInitializationTimeout` is 0.2 seconds. Before the fix, the test failed: zero GET requests in 10 seconds. This shows the hang.

    2. Cause, confirmed in Sources/MCP/Base/Transports/HTTPClientTransport.swift: `withThrowingTaskGroup` waits for ALL child tasks before it returns. `await signalTask.value` does not react to `cancelAll()`, and the continuation in `signalTask` only resumes when a session id arrives or on `disconnect()`. Thus the group could not return.

    3. GREEN: The timeout child task now calls `handleInitialSessionIDTimeout()` after the sleep. This function writes the timeout warning to the log, then resumes and clears `initialSessionIDContinuation` through the new `releaseInitialSessionIDSignal()`. `signalTask` completes, the group returns, and `connectToEventStream()` starts. Also removed the duplicated resume-and-clear code: `disconnect()` and `triggerInitialSessionIDSignal()` now call `releaseInitialSessionIDSignal()`.

    4. After the fix, the new test passes in 0.226 seconds (just after the 0.2 second timeout). Full `swift test`: 553 tests in 40 suites, all pass, no warnings.
  timestamp: 2026-08-29T13:49:10.325011+00:00
- actor: claude-code
  id: 01m16whgqrn1p2m3ry71h7sv2a
  text: |-
    ### implement — changed
    - evidence: 2 files — Sources/MCP/Base/Transports/HTTPClientTransport.swift, Tests/MCPTests/HTTPClientTransportTests.swift. RED: new test `testStandaloneGETStartsWhenNoSessionIDArrives` failed before the fix (0 GET requests in 10 s). GREEN: test passes in 0.226 s with `sseInitializationTimeout` = 0.2 s. Full `swift test`: 553 tests in 40 suites pass, 0 failures, 0 warnings.
    - next: ready for /review
  timestamp: 2026-08-29T13:49:57.880753+00:00
position_column: doing
position_ordinal: '80'
title: HTTPClientTransport streaming task hangs when no session id ever arrives
---
## What

`Sources/MCP/Base/Transports/HTTPClientTransport.swift`,
`startListeningForServerEvents()`. When the server never sends a session id,
the standalone GET never starts, and the documented
`sseInitializationTimeout` does not release it.

The wait uses a task group with two children:

- `try await Task.sleep(for: .seconds(timeout))`
- `await signalTask.value`

`group.next()` returns when the sleep completes, and `group.cancelAll()` runs.
But `withThrowingTaskGroup` awaits ALL children before it returns, and
`await task.value` on a non-throwing `Task` does not respond to cancellation:
it waits until `signalTask` completes. `signalTask` awaits a
`withCheckedContinuation` that only `triggerInitialSessionIDSignal()` or
`disconnect()` resumes. So when no session id ever arrives, the group never
returns and the streaming task hangs forever. No GET is ever issued.

## How it was found

Measured while writing the test for ^2zvezkm. A mock server whose POST
response carried no `Mcp-Session-Id` header made the transport issue zero GET
requests over ten seconds, with `sseInitializationTimeout` at 0.2 seconds.

## What to build

Make the timeout real: after `sseInitializationTimeout`, the streaming task
must proceed to `connectToEventStream()` even when the signal never fired.
One option: on timeout, resume and clear `initialSessionIDContinuation`, so
`signalTask` completes and the group returns. Use `/tdd`: a failing test with
`MockURLProtocol` whose POST response carries no session id, asserting a GET
arrives after the timeout.

## Acceptance Criteria

- [x] With no session id from the server, the standalone GET starts after
      `sseInitializationTimeout`.
- [x] A test covers it and fails before the fix.
- [x] `swift test` passes.

#transport #bug