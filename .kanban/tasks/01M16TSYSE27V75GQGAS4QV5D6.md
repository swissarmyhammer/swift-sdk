---
assignees:
- claude-code
position_column: todo
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

- [ ] With no session id from the server, the standalone GET starts after
      `sseInitializationTimeout`.
- [ ] A test covers it and fails before the fix.
- [ ] `swift test` passes.

#transport #bug