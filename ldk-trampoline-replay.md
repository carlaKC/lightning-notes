# Trampoline Replay after Restarts

Aiming to align trampoline work with the ongoing `ChannelManager`
project.

`from_channel_manager_data`:
- We have a `ChannelManagerData` struct that contains the fields
  that we've got saved in the channel manager. This contains things
  like `pending_claiming_payments`, but we're moving away from storing
  these.
- We read this from disk and then call `from_channel_manager_data`
- Look at `forward_htlcs` because this is something we've deprecated.
  - Looking at `claimable_payments` seems like it would have fit a bit
    better, but it hasn't been switched over yet.

When we restart we:
- Iterate through all the `FundedChannel` channels that we stored in the
  `ChannelManager`
  - Lookup its `ChannelMonitor`
    - If we have a `ChannelMonitor`:
      - If we have a stale `ChannelManager`, we're behind the
        `ChannelMonitor`; we don't know what we're missing so can't
        operationally continue and we close the channel.
      - Our channel tracks its pending `ChannelMonitor` updates; if
        the monitor ID that we've loaded has some of those we can just
        drop them, otherwise we keep them.
      - These blocked updates will need to be replayed to the
        `ChannelMonitor` to get it back up to state.
      - We insert the channel into our `per_peer_state`
    - If it's `awaiting_initial_mon_persist`: we _just_ created it and
      shut down while we were persisting the `ChannelMonitor`
    - If it's not found, then we failed to persist the `ChannelMonitor`,
      which is a fundamental violation of our contract (it must be
      persisted before we continue).

- Iterate through all `channel_monitors`
  - If we don't have a channel in `ChannelManager` and the channel
    has been closed, it puts it in `closed_channel_monitor_update_ids`
    to be handled appropriately

- Reload a bunch of data
  - `pending_outbound_payments` is directly read from disk and we
    restore it
  - Likewise for things like `fake_scid_rand_bytes`

- We have a set of `ChannelMonitorUpdate(s)` in
  `in_flight_monitor_updates` in the `ChannelManager`. We gave these
  to the `ChannelMonitor` but did not hear back so we don't know
  whether they were persisted.
  - If the `ChannelMonitor` has the update id, then we know that it
    was written and we can skip it.

- Go through each peer's state / each channel:
  -  Get the monitor for the channel that was persisted in
     `ChannelManager` and get any updates for them.
  - `handle_in_flight_updates` (1)

- Go through the remaining `in_flight_monitor_updates` (didn't have
  channels in `ChannelManager`, which is possible when we've closed a
  channel).
  - We create a per-peer state just for this monitor update and insert
    it
  - `handle_in_flight_updates` (1)

(1) `handle_in_flight_updates`:
- Checks if the `ChannelMonitor` already has all the updates
- Replays all update ids that are > `ChannelMonitor`'s last update

Finally if we're in "reconstruct from monitors world", we'll start to
re-build the `ChannelManager`'s state from the `ChannelMonitor`s.

`pending_claims_to_replay`:
- Tracks preimages for forwarding HTLCs that need to be applied on the
  incoming edge.

`already_forwarded_htlcs`:
- If we have an inbound HTLC that looks like it's already been
  forwarded on the outgoing link, we track it here.
- It should be:
  - Present on the outbound edge's monitor
  - Removed from the outbound edge via claim
  -> If it's in neither of these states, we assume that it was failed
     on the outgoing edge.
- We look at `inbound_forwarded_htlcs`  in the channel and match those
  that are `InboundUpdateAdd::Forwarded`, adding to our map of
  `already_forwarded_htlcs`
- If the channel is closed:
  - Get all the outbound htlcs on the channel (incl source)
  - `insert_from_monitor_on_startup` will populate our payments,
    we just don't attempt any retries

Q: difference between reading `Channel` and `ChannelManager`
- When we get to this stage of the reconstruction, we only have channels
  that are in sync with monitors (we've closed those who are behind, and
  replayed updates for those that are ahead!).

- Next we iterate through all `channel_monitors`
  - Get the channel and peer state
  - Go through the `Channel`'s `outbound_htlc_forwards`
    - Everything that's got a "forwarding" source in both the holding
      cells and in `pending_outbound_htlcs`
  - Remove the htlc from `decode_update_add_htlcs` if already on channel
  - Remove the htlc from `already_forwarded_htlcs` if already on channel

- If the channel is closed, go through all `current_outbound_htlcs`
  - If it's a `PreviousHopData`
    - De-dup `decode_update_add_htlcs` and `already_forwarded_htlcs`
      as before
  - If it's a `OutboundRoute` we'll `claim_htlc` if we have a preimage
  - Next we go through `onchain_failed_outbound_htlcs` and push to
    our `failed_htlcs`

- Regardless of closed or not, go through `get_all_current_outbound_htlcs`:
  - If it's `PreviousHopData`:
    - If it has a preimage, push into `pending_claims_to_replay`

- Process all of our outbound_scid_alias-es

- If reconstructing, we de-dup all of our `decode_update_add_htlcs`
  and `already_forwarded_htlcs` (if they're in our failed_htlcs or
  `claimable_payments`, we don't need to handle them).

Create a channel manager!

- Go through all our `channel_monitors`:
  - Get all of the outbound htlcs on the monitor
  - Match on their source:
    - If reconstructing, remove any forwarded htlcs from
      `decode_update_add_htlcs`
    - If the channel is still open and we're not reconstructing from
      monitors:
      - Dedups `decode_update_add_htlcs`
      - Removes from `forward_htlcs` if already forwarded
      - Removed from `pending_intercepted` if already forwarded
    - For outbound payments, calls `claim_htlc` if there's a preimage
      present

  - Go through all our failed outbound htlcs
    - Push to `failed_htlcs`

  - Go through all our outbound htlcs
    - Get the previous_htlcs associated with them
    - Lookup the inbound montior
    - Check whether we need to reply the htlc
    - Push into `pending_claims_to_replay` if necessary

- If reconstructing:
  - Remove any failed htlcs from `decode_update_add_htlcs`
  - Likewise remove any failed htlcs from `claimable_payments`

- Create:
  - `decode_update_add_htlcs`: decoded_update_add_htlcs{_legacy}
  - `forward_htlcs`: (empty for reconstruction)
  - `pending_intercepted_htlcs`: (empty for reconstruction)
-> These fields are used to populate our channelmanger's fields
-> When we're restoring from `ChannelMonitor`, we'll replay all of our
  un-forwarded htlcs via `decode_update_add_htlcs` to re-build our state
  maps. We *definitely* can't have already forwarded htlcs in here,
  because we'd double forward them.

Q: What are we actually persisting in `ChannelManager` at present?
- We're still writing everything, but we're optionally restoring from
  `ChannelMonitor` when we want.

Trampoline thoughts:
- We don't have the information we need in
  `InboundUpdateAdd::Forwarded` to restore a trampoline payment?
  Where would these be if they're waiting on accumulation?
  What if they're done accumulating but not dispatched yet?
- `dedup_decode_update_add_htlcs` checks based on previous hop htlc
  ID (we're going to have many for trampoline):
  -> That's okay, if we have a trampoline payment on the outbound
  monitor, then we can safely assume that the 
- If we can replay all of our trampoline HTLCs (that haven't been
  forwarded) through `decode_update_add_htlcs`, then we can rebuild
  the state of our `awaiting_trampoline_forwards` through this?

- [ ] Probably also need to replay claim of trampoline htlc in
  `handle_in_flight_updates`

Our `pending_outbounds`:
- We start with values that have been written in the `ChannelManager`.
- If the channel has been closed, we insert a `pending_outbound` for the
  trampoline payment (like we do for `OutboundRoute`).

Deduplicating `decode_update_add_htlcs` and `already_forwarded_htlcs`:
- We store _every_ inbound htlc in our `HTLCSource`, so will be able
  to correctly de-duplicate by removing each inbound one at a time by
  htlc id.

`handle_monitor_update`:
- `handle_new_monitor_update_with_status`
  - `handle_new_monitor_update_locked_actions_handled_by_caller`:
    - If we can push an event, `update_channel`
    - Otherwise, push to `pending_background_events`
  - If we complete all of our updates:
    - `try_resume_channel_post_monitor_update`:
      - If there are no `blocked_monitor_updates_pending`:
        - `monitor_updating_restored`:
          - Progresses state of channel
          - Add outbound payments to: `committed_outbound_htlc_sources`
        - `handle_channel_resumption`:
          - Updates channelmanager's state accordingly
          - Queues messages and events

`handle_post_monitor_update_chan_resume`:
- `post_monitor_update_unlock`:
  - We have completed update of our `ChannelMonitor`
  - We have dropped per-per state locks in `ChannelManager`
  - `handle_monitor_update_completion_actions`:
    - If we have a `PaymentClaimed`:
    - If we've got a `pending_claiming_payments`:
      - Push an event
    - Otherwise emit event if required and free channel
      - `handle_monitor_update_release`:
        - Restore channel to regular operation once an event has been
          processed
  - Forward any HTLCs that need forwarding
  - Finalize any claims any that need handling
  - `prune_persisted_inbound_htlc_onions`:
    - Updates inbound channel state to reflect that we're now in a
      forwarded state

## Trampoline Replay

I'm still getting familiar with the `ChannelMonitor` restore project, but I think that the following changes would be necessary for trampoline:

### Adding + Forwarding Trampoline

Trampoline committed:
- `ChannelMonitor`(inbound): wrote `LatestHolderCommitmentTXInfo` and
  `CommitmentSecret` for the commitment containing the HTLC.
- `ChannelManager`: wrote `pending_inbound_htlcs` with
  `InboundHTLCState::Committed` and `InboundUpdateAdd::WithOnion`.

The HTLC will be added to `decode_update_add_htlcs` when it is returned
by `inbound_htlcs_pending_decode`. Since the HTLC has not yet been
forwarded, it will correctly be replayed through
`process_pending_update_add_htlcs`. This will add the HTLC to our
`awaiting_trampoline_forwards` where MPP parts will be accumulated and
a payment will be dispatched when they're all there.

Trampoline forward processing:
- `ChannelMonitor`: no update.
- `ChannelManager`: processing pipeline removed the HTLC from
  `forward_htlcs` and adds it to `pending_outbound_htlcs` on the
  outgoing channel. We also create a `pending_outbound_payment` for the
  trampoline payment.

If we go down before the `ChannelManager` is written, the trampoline
will be replayed via `decode_update_add_htlcs` as before, and we'll
re-create the entry in `pending_outbound_payments`.

Trampoline forward dispatched:
- `ChannelMonitor`(outbound): wrote `LatestCounterpartyCommitmentTXInfo`
- `ChannelManager`: written with the `LocalAnnounced` in
  `pending_outbound_htlcs`, the inbound HTLC still in 
  `InboundUpdateAdd::WithOnion` and an entry in
  `pending_outbound_payments`.

If we go down after this step, we'll re-load the incoming HTLC into
our `decode_update_add_htlcs` but then de-duplicate it because it's
present in our `pending_outbound_htlcs` on the outbound channel. This
will work for the case where we have multiple inbound trampoline HTLCs
because each outbound `HTLCSource` lists *all* the inbound HTLCs
associated with it, so we'll remove them all (with some duplication).

- `ChannelMonitor`(outbound): wrote `CommitmentSecret`.
- `ChannelManager`: written with outbound HTLC in `Committed` state,
  inbound HTLC in `InboundUpdateAdd::Forwarded`.

If we go down after this step, our inbound HTLC will now be returned
as part of `inbound_forwarded_htlcs`, so it will be tracked in our
`already_forwarded_htlcs` and then pruned by the presence of a matching
HTLC in `pending_outbound_htlcs`.

### HTLC Settle

By this stage our inbound HTLC is recorded as
`InboundUpdateAdd::Forwarded`, so we're not at risk of a duplicate
forward. It is added in `already_forwarded_htlcs` on restart.

- `ChannelMonitor`(outbound): wrote commitment where HTLC is claimed.
- `ChannelManager`: outbound HTLC updated to `RemoteRemoved`, and the
  `pending_outbound_payment` is marked as `Fulfilled`.

If we go down here, `get_all_current_outbound_htlcs` will return the
`HTLCSource` along with its preimage and replay the claim to all of the
inbound monitors that need it. Because we store *all* inbound HTLCs
with *any* outbound trampoline HTLC, we'll always replay the preimage
to all of the inbound monitors correctly.

We prune the `already_forwarded_htlcs` entry because
`outbound_htlc_fowards` still contains the outgoing htlc.

Assuming that we have blocked updates on the inbound channel and can't
update commitment yet:
- `ChannelMonitor`(inbound): wrote `PaymentPreimage`
- `ChannelManager`: unchanged

As above, we'll handle preimage replays and de-dup
`already_forwarded_htlcs`.

- `ChannelMonitor`(outbound): wrote `CommitmentSecret`
- `ChannelManager`: HTLC is removed from `pending_outbound`.

If we go down here, the HTLC is no longer on our outbound monitor or
in the manager but it is still present on the incoming channel. We
will remove the HTLC from `already_forwarded_htlcs` when we process
our claim replays because we have the preimage on the inbound monitor.

The same is true for updates to the inbound monitor to settle the
incoming HTLC back.

### HTLC Fail

By this stage our inbound HTLC is recorded as
`InboundUpdateAdd::Forwarded`, so we're not at risk of a duplicate
forward. It is added in `already_forwarded_htlcs` on restart.

- `ChannelMonitor`(outbound): `LatestHolderCommitmentTxInfo` without
  the HTLC.
- `ChannelManager`: outbound HTLC is updated to
  `AwaitingRemoteRevokeToRemove`

If we go down here, the HTLC will be returned by
`outbound_htlc_forwards`, so it will de-dup `already_forwarded_htlcs`.

- `ChannelMonitor`(outbound): wrote `CommitmentSecret`
- `ChannelManager`: HTLC is removed from `pending_outbound`, and we
  remove the payment from `pending_outbound_payments`.

If we go down here, the HTLC will no longer be in our outbound monitor
or `pending_outbound_htlcs`. It is still added to
`already_forwarded_htlcs`, and will not be pruned so we'll proceed
to fail it back at the end of our replays. We'll successfully fail
back all of our incoming HTLCs for trampoline as they're all listed
in `HTLCSource`.

This failure is piped through `fail_htlc_backwards_internal`, but our
trampoline payment is no longer present. We'll just fail the not-found
payment back with the error we're provided with.

- `ChannelMonitor`: `LatestCounterpartyCommitmentTXInfo`
- `ChannelManager`: inbound HTLC in `LocalRemoved`

As above, we fall all the way through to failure via
`already_forwarded_htlcs`.

- `ChannelMonitor`(inbound): `CommitmentSecret`
- `ChannelManager`: inbound HTLC is removed

We're done!
