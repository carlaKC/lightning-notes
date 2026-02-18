# Channel Monitor

Walkthrough as of: faf7b8aa0

`ChannelMonitor`:
- Handles chain events and generates on-chain transactions required
  to ensure that no funds are lost.
  (This is similar to `ChannelArbitrator` in LND, if you're from there).
- Holds `ChannelMonitorImpl` in a mutex, most of the methods that it
  implements hold the lock and cut straight through to the inner impl.

Our `ChannelManagerReadArgs` holds a `HashMap<ChannelId, ChannelMonitor>`:
- `channel.rs`:
  - `commitment_signed`:
    - `initial_commitment_signed` returns a `ChannelMonitor`
  - `funding_signed`: returns a `ChannelMonitor`
-> These are passed to `watch_channel` (`Watch` trait)

`ChainMonitor` holds a `HashMap<ChannelId, ChannelMonitor>`, and
implements the `Watch` trait that is responsible for watching the
on-chain activity of channels.

- `watch_channel`:
  - `persist_new_channel`: writes new channel to disk.
  - If we can watch outputs, add channel outputs to filter:
    - Funding outpoint
    - Counterparty commitment, if on chain
  - Adds `ChannelMonitor` to the `HashMap` in the `ChainMonitor`.
    - `pending_monitor_updates` includes the latest update ID

This essentially reports the channel for tracking, adding it to our
set of channels being monitored.

- `update_channel`
  - Look up our `channel_id` in the `HashMap`, fail if not found.
  - Get all `pending_monitor_updates` for the channel
  - Call `update_monitor`
    - TODO! RESUME HERE!
  - Call `update_persisted_cahnnel`
  - If we can watch outputs, register the funding outpoint

- `release_pending_monitor_events`

## Monitor / Manager 

- `ChannelMonitor` *must* be written on `update_channel` (either on
  return, or on completion of asynchronous write).
- `ChannelManager` writes happen out of bound, and it is frozen while
  the write is happening.
- These can be out of sync, and we'll force close channels if what
  we have in `ChannelMonitor` doesn't match `ChannelManager`.

We coordinate with the `update_id` for our `ChannelMonitor`.
- If they're equal, great - we're in sync.
- If `ChannelManager` > `ChannelMonitor`:
- If `ChannelManager` < `ChannelMonitor`: the channel manager doesn't
  know what it missed and can't replay anything so it'll force close
  the channel (all that info is in `ChannelMonitor`, and safe to use).
