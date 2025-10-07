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



## Outstanding questions

Q: Why does `ChannelManager` need to have a map of monitors, we only
really use them in `read`?
