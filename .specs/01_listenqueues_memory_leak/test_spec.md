# Test Specification: listenQueues Memory Leak Fix

## Overview

Tests validate that `listenQueues` entries are cleaned up on listener disconnect and that existing dial-listen communication is preserved. Tests are written for Go's `testing` package within the `service` package (white-box access to `listenQueues`).

## Test Cases

### TS-01-1: Entry Removed on Context Cancellation

**Requirement:** 01-REQ-1.1
**Type:** unit
**Description:** Verifies that the `listenQueues` entry is removed when `Listen()` returns due to context cancellation.

**Preconditions:**
- A `ControllerService` instance with an empty `listenQueues`.
- A mock gRPC stream with a cancellable context.
- Authentication and lease validation pass.

**Input:**
- Call `Listen()` with a valid lease name, then cancel the context.

**Expected:**
- After `Listen()` returns, `listenQueues.Load(leaseName)` returns `(nil, false)`.

**Assertion pseudocode:**
```
ctx, cancel = context.WithCancel(background)
go service.Listen(req, mockStream{ctx})
cancel()
// wait for Listen to return
_, loaded = service.listenQueues.Load(leaseName)
ASSERT loaded == false
```

### TS-01-2: Entry Removed on Send Failure

**Requirement:** 01-REQ-1.2
**Type:** unit
**Description:** Verifies that the `listenQueues` entry is removed when `Listen()` returns due to `stream.Send` failure.

**Preconditions:**
- A `ControllerService` instance with an empty `listenQueues`.
- A mock gRPC stream whose `Send()` returns an error.
- Authentication and lease validation pass.

**Input:**
- Call `Listen()` with a valid lease name, send a message to the channel, `Send()` fails.

**Expected:**
- After `Listen()` returns, `listenQueues.Load(leaseName)` returns `(nil, false)`.

**Assertion pseudocode:**
```
mockStream.Send = func(resp) error { return errors.New("send failed") }
go service.Listen(req, mockStream)
queue, _ = service.listenQueues.Load(leaseName)
queue.(chan) <- &pb.ListenResponse{}  // trigger Send
// wait for Listen to return
_, loaded = service.listenQueues.Load(leaseName)
ASSERT loaded == false
```

### TS-01-3: Dial-Listen Communication Works

**Requirement:** 01-REQ-2.1
**Type:** unit
**Description:** Verifies that `Dial()` messages are delivered to an active `Listen()` stream.

**Preconditions:**
- A `ControllerService` instance.
- A mock gRPC stream that records sent messages.
- `Listen()` is actively running for the target lease.

**Input:**
- Call `Dial()` to send a `ListenResponse` for the lease.

**Expected:**
- The mock stream's `Send()` receives the `ListenResponse` with matching `RouterEndpoint` and `RouterToken`.

**Assertion pseudocode:**
```
go service.Listen(req, mockStream)
// wait for Listen to be ready
service.Dial(ctx, dialReq)
// wait for message delivery
ASSERT mockStream.lastSent.RouterEndpoint != ""
ASSERT mockStream.lastSent.RouterToken != ""
```

### TS-01-4: Buffered Channel Capacity

**Requirement:** 01-REQ-2.2
**Type:** unit
**Description:** Verifies that the listen queue channel has capacity 8.

**Preconditions:**
- A `ControllerService` instance.

**Input:**
- Trigger `LoadOrStore` for a lease name.

**Expected:**
- The channel has capacity 8.

**Assertion pseudocode:**
```
queue, _ = service.listenQueues.LoadOrStore("test-lease", make(chan *pb.ListenResponse, 8))
ASSERT cap(queue.(chan *pb.ListenResponse)) == 8
```

## Edge Case Tests

### TS-01-E1: Concurrent Dial and Listen Cleanup No Panic

**Requirement:** 01-REQ-1.E1
**Type:** unit
**Description:** Verifies that concurrent `Dial()` sends and `Listen()` cleanup do not panic or deadlock.

**Preconditions:**
- A `ControllerService` instance.
- Multiple goroutines performing `LoadOrStore`, channel send, and `Delete` concurrently.

**Input:**
- Run 100 iterations of concurrent `LoadOrStore` + send + `Delete` on the same key.

**Expected:**
- No panic, no deadlock, completes within timeout.

**Assertion pseudocode:**
```
FOR i IN 1..100:
    go func():
        queue, _ = service.listenQueues.LoadOrStore(key, make(chan *pb.ListenResponse, 8))
        select:
            case queue.(chan) <- &pb.ListenResponse{}:
            default:
        service.listenQueues.Delete(key)
ASSERT completes within 5 seconds
ASSERT no panic
```

### TS-01-E2: Re-Listen After Cleanup Creates Fresh Channel

**Requirement:** 01-REQ-1.E2
**Type:** unit
**Description:** Verifies that a new `Listen()` after cleanup creates a new channel.

**Preconditions:**
- A `ControllerService` instance.
- A previous `Listen()` has completed and cleaned up.

**Input:**
- Call `Listen()`, cancel context (cleanup), then call `Listen()` again.

**Expected:**
- Second `Listen()` gets a new channel. Messages sent to it are received.

**Assertion pseudocode:**
```
// First listen + cleanup
ctx1, cancel1 = context.WithCancel(background)
go service.Listen(req, mockStream{ctx1})
cancel1()
// wait for cleanup

// Second listen
ctx2, cancel2 = context.WithCancel(background)
go service.Listen(req, mockStream2{ctx2})
queue, _ = service.listenQueues.Load(leaseName)
ASSERT queue != nil  // new channel exists
queue.(chan) <- &pb.ListenResponse{RouterEndpoint: "test"}
// wait for delivery
ASSERT mockStream2.lastSent.RouterEndpoint == "test"
cancel2()
```

### TS-01-E3: Dial to Removed Channel No Panic

**Requirement:** 01-REQ-1.E3
**Type:** unit
**Description:** Verifies that sending to a channel reference obtained before `Delete` does not panic.

**Preconditions:**
- A channel reference obtained via `LoadOrStore`.
- The map entry is then deleted.

**Input:**
- Send to the channel after the map entry is deleted.

**Expected:**
- No panic. The send either succeeds (buffered) or blocks (handled by select with ctx.Done).

**Assertion pseudocode:**
```
queue, _ = service.listenQueues.LoadOrStore(key, make(chan *pb.ListenResponse, 8))
service.listenQueues.Delete(key)
// sending to a deleted-from-map channel is fine — it's just a Go channel
select:
    case queue.(chan) <- &pb.ListenResponse{}:
        ASSERT true  // buffered send succeeded
    default:
        ASSERT true  // buffer full, also fine
ASSERT no panic
```

## Property Test Cases

### TS-01-P1: Cleanup Guarantee

**Property:** Property 1 from design.md
**Validates:** 01-REQ-1.1, 01-REQ-1.2, 01-REQ-1.3
**Type:** property
**Description:** For any `Listen()` execution that reaches `LoadOrStore`, the entry is removed when `Listen()` returns.

**For any:** Return reason (context cancellation, send error), lease name (arbitrary string)
**Invariant:** After `Listen()` returns, `listenQueues.Load(leaseName)` returns `loaded == false`.

**Assertion pseudocode:**
```
FOR ANY leaseName IN arbitrary_strings, exitMode IN {cancel_ctx, send_error}:
    setup Listen() with leaseName
    trigger exit via exitMode
    wait for return
    _, loaded = service.listenQueues.Load(leaseName)
    ASSERT loaded == false
```

### TS-01-P2: Concurrent Safety

**Property:** Property 2 from design.md
**Validates:** 01-REQ-1.E1, 01-REQ-1.E3
**Type:** property
**Description:** For any interleaving of `LoadOrStore`, channel send, and `Delete` operations, the system does not panic or deadlock.

**For any:** Number of concurrent goroutines (1-50), operation ordering (random)
**Invariant:** All goroutines complete without panic within a timeout.

**Assertion pseudocode:**
```
FOR ANY n IN 1..50:
    launch n goroutines each doing random sequences of:
        LoadOrStore, channel send (non-blocking), Delete
    ASSERT all complete within 10 seconds
    ASSERT no panic recovered
```

### TS-01-P3: Functional Preservation

**Property:** Property 3 from design.md
**Validates:** 01-REQ-2.1, 01-REQ-2.2
**Type:** property
**Description:** For any message sent via `Dial()` while `Listen()` is active, the message is delivered to the stream.

**For any:** Valid `ListenResponse` message
**Invariant:** The message is received by the mock stream's `Send()`.

**Assertion pseudocode:**
```
FOR ANY response IN valid_ListenResponses:
    setup active Listen()
    send response to queue
    ASSERT mockStream received response
```

### TS-01-P4: Re-listen Correctness

**Property:** Property 4 from design.md
**Validates:** 01-REQ-1.E2
**Type:** property
**Description:** For any number of sequential listen-cleanup-relisten cycles, each new `Listen()` creates a functional channel.

**For any:** Number of cycles (1-10)
**Invariant:** After each cleanup, a new `Listen()` successfully receives messages.

**Assertion pseudocode:**
```
FOR ANY cycles IN 1..10:
    FOR i IN 1..cycles:
        start Listen(), cancel, wait for cleanup
        ASSERT listenQueues.Load(leaseName) == (nil, false)
    start Listen()
    send message to queue
    ASSERT message delivered
```

## Coverage Matrix

| Requirement | Test Spec Entry | Type |
|-------------|-----------------|------|
| 01-REQ-1.1 | TS-01-1 | unit |
| 01-REQ-1.2 | TS-01-2 | unit |
| 01-REQ-1.3 | TS-01-P1 | property |
| 01-REQ-1.E1 | TS-01-E1 | unit |
| 01-REQ-1.E2 | TS-01-E2 | unit |
| 01-REQ-1.E3 | TS-01-E3 | unit |
| 01-REQ-2.1 | TS-01-3 | unit |
| 01-REQ-2.2 | TS-01-4 | unit |
| 01-REQ-2.E1 | TS-01-E1 | unit |
| Property 1 | TS-01-P1 | property |
| Property 2 | TS-01-P2 | property |
| Property 3 | TS-01-P3 | property |
| Property 4 | TS-01-P4 | property |
