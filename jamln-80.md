# Decaying Average Issues

Running into issue with sped-up time and decaying average going backwards.

To debug this, created a unit test with 2x tasks - each locking and
updating a decaying average.

Task 1 picks a time:
- Task 1: picked now: Instant { tv_sec: 81021, tv_nsec: 865996166 }

Task 2 updates the value in the decaying average:
- Task 2: updated avg to Instant { tv_sec: 81024, tv_nsec: 425371166 }
- Task 2: picked now: Instant { tv_sec: 81024, tv_nsec: 476413166 }

Task 1 errors out
- Task 1: Err: last updated reputation at Instant { tv_sec: 81024, tv_nsec: 425371166 }, read at Instant { tv_sec: 81021, tv_nsec: 865996166 }
- Task 1: triggered for shutdown

I suspect that this may have to do with the yield/wait of an async
task?
- When I have a tokio lock, when we await on the locking it yields
  control to tokio to decide who gets control next.
- We pick the timestamp before we yield to tokio, and set it after
- If there's another HTLC that sneaks in front of ours, then its
  timestamp will be after ours
- When we get back control, our timestamp is stale and the decaying
  average thinks we're going backwards.

Tests are in: 80-test-race-timestamps.

## Real Code Check

These races happen when we `add_htlc` and `remove_htlc` (since this
is where we update the decaying average).

- `intercept_htlc`:
- Picks `added_ins` 
- Calls `inner_add_htlc`
  - Gets `network_nodes.lock`
    - Calls `forward_manager.add_htlc`
      - Gets `forward_manager`.inner_lock
        - Updates reputation

- `notify_resolution`:
  - Picks `resolved_ins`
  - Calls `inner_resolve_htlc`
    - Gets `network_nodes.lock`
      - Calls `resolve_htlc`
        - Gets `inner_lock`
          - Updates reputation

Once we hold `network_nodes`, no other node in the network can be
updated in that instant - the `forward_manager` lock is _actually_
irrelevant. *If* we get our timestamp after the lock is acquired,
we shouldn't have a problem?

(the `forward_manager` lock exists for the sake of the library)

