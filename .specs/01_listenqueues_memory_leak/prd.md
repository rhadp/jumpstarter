# PRD: Fix Memory Leak in listenQueues on Listener Disconnect

**Source:** [jumpstarter-dev/jumpstarter#362](https://github.com/jumpstarter-dev/jumpstarter/issues/362)

## Problem

In `controller/internal/service/controller_service.go`, the `Listen()` method creates an entry in the `listenQueues` `sync.Map` via `LoadOrStore` (line 442). When the listener disconnects — either through context cancellation (line 445) or `stream.Send` failure (line 449) — the function returns without removing the entry.

There is no `listenQueues.Delete()` call anywhere in the codebase, so channels accumulate in the map over time as leases are created and listeners disconnect. This is a memory leak.

## Context

- `Listen()` is called by an exporter to receive dial notifications for a specific lease. It loops reading from a buffered channel (capacity 8) and forwarding messages to a gRPC stream.
- `Dial()` is called by a client. It creates a router token/endpoint pair and sends it to the same channel via `LoadOrStore`, notifying the exporter of a new connection.
- Both functions use `LoadOrStore` so either side can create the channel first.
- There is a 1:1 relationship between active leases and listeners.

## Suggested Fix

Add `defer s.listenQueues.Delete(leaseName)` after the `LoadOrStore` call in `Listen()`, so the entry is removed when the listener disconnects.

## Clarifications

1. **Race between Delete and Dial**: After `Listen()` deletes the entry, if `Dial()` calls `LoadOrStore`, it atomically creates a new channel. If a subsequent `Listen()` arrives, it gets that channel via `LoadOrStore` — messages are received correctly. The `sync.Map` atomicity handles this.
2. **Pending message discard**: When the listener disconnects, pending messages in the buffered channel are discarded. This is acceptable because the router tokens have a 30-minute expiry and the lease session is effectively over.
3. **No channel close needed**: `Dial()` uses `select` with `ctx.Done` when sending, so it won't block forever. Closing the channel with concurrent senders would cause a panic.
4. **One listener per lease**: Architecturally, one exporter listens per lease. `LoadOrStore` reuses the existing channel if one exists, confirming this assumption.

## Scope

This is a targeted bug fix. Only `Listen()` in `controller_service.go` needs modification.
