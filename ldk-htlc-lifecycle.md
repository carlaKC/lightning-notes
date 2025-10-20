# HTLC Lifecycle

Full commitment dance for adding a HTLC that's intended to be forwarded.

## Messaging
Throughout this walkthrough, messages are pushed to the `ChannelManager`'s
`per_peer_state.pending_msg_events`. They are delivered to the remote peer
as follows:

The `ChannelManager` implements `BaseMessageHandler`:
- `get_and_clear_pending_msg_events`: returns a vec of messages
- `ChannelMessageHandler` extends `BaseMessageHandler`
- `MessageHandler` has a `chan_handler` impl of `ChannelMessageHandler`
- `process_events` pulls the messages and queues them for the peer

When messages are received, they are also piped to the 
`ChannelMessageHandler`, which delivers the incoming messages to the
channel message handler.

## Start State
Starting state for channel:
- When we receive a `channel_ready` and we've sent our own, we'll
  progress our `ChannelState` enum on `Channel` to `ChannelReady`
  - Pull all flags from previous state and set `FundedStateFlags::ALL`

## Adding a HTLC
- Once a channel has been fully opened, it's going to be tracked in
  the `ChannelManger`'s `per_peer_state`:
  - `channel_by_id` will have `ChannelPhase::Funded(Channel)`
- When the `PeerManager` gets an `update_add_htlc`, it'll call 
  `handle_update_add_htlc` on its `ChannelMessageHandler` (which is
  implemented by `ChannelManger`
- `internal_update_add_htlc` returns an indication of whether we need
  to do any persistence after this operation:
  - If something has gone so wrong that we have to close the channel,
    then we notify that persistence is required
  - SkipPersistHandleEvents:
  - SkipPersistNoEvents:
- `ChannelManager`/`internal_update_add_htlc`:
- `Channel` / channel.update_add_hltc
  - Performs various state/value sanity checks and runs
    `validate_update_add_htlc` on each pending funding state
  - Push a `InboundHTLCOutput` to `pending_inbound_htlcs` with
    `RemoteAnnounced(InboundHTLCResoultion::Pending)` state

```
Incoming Channel:
  pending_inbound_htlcs:
    state: InboundHTLCState::RemoteAnnounced(
       InboundHTLCResolution::Pending(update_add_htlc)
    )
```

## Receiving a Commitment
- When the `PeerManager` gets a `commitment_signed`, it will call
  `handle_commitment_signed` on its `ChannelMessageHandler` (which is
  implemented by `ChannelManager`)
- We `notify_on_drop` the `PersistenceNotifier`, indicating that we 
  must persist once this function is done
- `internal_commitment_signed`:
  - Fetches the channel and asserts that it's reached the funded stage
  - `try_channel_entry` with `commitment_signed`
    - Assuming that the channel is funded, and we aren't splicing,
      we call `funded_channel.commitment_signed`:
      - `validate_commitment_signed`:
        - Sanity checks on the channel's state that an update is possible
          `build_commitment_transaction`:
            - `local=true`
            - `generated_by_local=false`
            - HTLC is in `pending_inbound_htlcs` with state
              `RemoteAnnounced` so we'll call `add_htlc_output`:
              - `outbound=false`
              - Added to `included_non_dust_htlcs`: 
              - Included in the commitment transaction
          - Validates signatures
     - Creates `ChannelMonitorUpdateStep::LatestHolderCommitmentTXInfo`
     - `commitment_signed_update_monitor(^)`:
       - For each `pending_inbound_htlcs`:
         - Update state to `AwaitingRemoteRevokeToAnnounce`
         - `need_commitment = T`
       - We don't have any pending outgoing HTLCs at this point, so
         there's no action for `pending_outbound_htlcs`
       - Create a `ChannelMonitorUpdate` that has the new commitment info
       - Update `expecting_peer_commitment_signed` = false
       - Update `resend_order` = `RAACommitmentOrder::CommitmentFirst`

```
Incoming Channel:
  pending_inbound_htlcs:
    state: InboundHTLCState::AwaitingRemoteRevokeToAnnounce

expecting_peer_commitment_signed: false
resend_order: RAACommitmentOrder::CommitmentFirst
is_awaiting_remote_revoke: false
```

   - Assuming that `!is_monitor_update_in_progress`
   - `need_commitment` && `!is_awaiting_remote_revoke`:
    - `build_commitment_no_status_check`:
      - Upgrades HTLCs from `AwaitingRemoteRevokeToAnnounce` to 
        `AwaitingAnnouncedRemoteRevoke`
      - `self.resend_order` = `RevokeAndAckFirst`
      - Creates `LatestCounterpartyCommitmentTXInfo` update
      - `self.awaiting_remote_revoke` = T 

```
Incoming Channel:
  pending_inbound_htlcs:
    state: InboundHTLCState::AwaitingAnnouncedRemoteRevoke

expecting_peer_commitment_signed: false
resend_order: RAACommitmentOrder::RevokeAndAckFirst
is_awaiting_remote_revoke: true

```

  - Append update to `monitor_update`
  - `monitor_updating_paused`(true, true, false, {empty vecs}:
    - There are a bunch of state-machine related variables here:
      - `self.pending_revoke_and_ack` = true
      - `self.pending_commitment_signed` = true
	  - `self.pending_channel_ready` = false
      - `self.pending_forwards/failures/finalized` = empty
	  - `self.update_in_progress` = true
      - `self.monitor_update_in_progress` = true

```
Incoming Channel:
  pending_inbound_htlcs:
    state: InboundHTLCState::AwaitingAnnouncedRemoteRevoke

expecting_peer_commitment_signed: false
resend_order: RAACommitmentOrder::RevokeAndAckFirst
is_awaiting_remote_revoke: true

monitor_pending_revoke_and_ack = true
monitor_pending_commitment_signed = true
monitor_pending_forwards = []
monitor_pending_failures = []
monitor_pending_finalized_fulfills = []
monitor_monitor_update_in_progress = true
```

  - Update is pushed onto the channel's queue:
    - If there are blocked updates, the user can't handle it now
    - If there's nothing queued, then return for the user to persist
    - These are stored in `blocked_monitor_updates` on the channel

In `internal_commitment_signed` we get `(None, LatestHolderCommitment)`
from `chan.commitment_signed`:
- `handle_new_monitor_update!`:
  - Gets `in_flight_monitor_updates`
  - Call `chain_monitor.update_channel`
      - Gets the specific `chain_monitor` and acquires its lock:
        - `chain_monitor.update_monitor`:
          - Gets the `monitor` for the channel
          - `update_monitor(update)` / `inner.update_monitor`:
            - Checks that state is okay
              - `LatesttHolderCommitmentTxInfo`
                - `provide_latest_holder_commitment_tx`:
                - Reports our latest tx to the `onchain_tx_handler`
                - Updates its view of the current commitment tx
              - `LatestCounterpartyCommitmentTXInfo`
                - `provide_latest_counterparty_commitment_tx`
                - Likewise notifies the chain monitor of new tx
      - If there are no updates pending on channel monitor:
        - `monitor_updating_restored` returns updates:
          - `pending_revoke_and_ack` = `true`; create `revoke_and_ack`
          - `pending_commitment_signed` = `true`; create `commitment_signed`
          - `signer_pending_revoke_and_ack=false`, so don't set 
            `signer_pending_commitment_update`
          - Update state machine:
            - `pending_revoke_and_ack` = F
            - `pending_commitment_signed` = F
          - Returns a set of `MonitorRestoreUpdates` with a `RAA` and
            `CommitmentSigned` it does not have any HTLCs on it because 
            the monitor's various `pending_forwards/failures/updates_adds`
            are not yet set
```
Incoming Channel:
  pending_inbound_htlcs:
    state: InboundHTLCState::AwaitingAnnouncedRemoteRevoke

expecting_peer_commitment_signed: false
resend_order: RAACommitmentOrder::RevokeAndAckFirst
is_awaiting_remote_revoke: true

monitor_pending_revoke_and_ack = false
monitor_pending_commitment_signed = false
monitor_pending_forwards = []
monitor_pending_failures = []
monitor_pending_finalized_fulfills = []
monitor_monitor_update_in_progress = true
signer_pending_commitment_update = false
```

Back in `handle_new_monitor_update!`:
- Call `handle_channel_resumption`:
  - We don't have any HTLCs that we need to process yet (because the
    update was empty).
  - Push `SendRevokeAndAck` message
  - Push `SendCommitmentSigned`
  - There aren't any HTLCs to handle at this stage
- `handle_monitor_update_completion_actions`
- There aren't any `htlc_forwards` or `decode_update_adds` returned

Once we've sent our `RevokeAndAck` and `CommitmentSigned` to the peer,
we expect them to send a `RevokeAndAck` back to us, which will
irrevocably commit to the HTLC.

## Receiving Revoke and ACK

- When the `PeerManager` gets a `revoke_and_ack`, it will call 
  `handle_revoke_and_ack` on its `ChannelMessageHandler` (which is
  implemented by `ChannelManager`
- We `notify_on_drop` the `PersistenceNotifier`, indicating that we
  must persist once this function is done
- `internal_revoke_and_ack`:
  - Looks up peer/channel as above, and asserts that the channel is in
    the correct state
  - `try_chan_phase_entry` with `channel.revoke_and_ack` 
    - Check that revocation point is as expected for previous commit
    - Create `ChannelMonitorUpdate` with a `CommitmentSecret` update
    - Clear state on `Channel`:
      - `awaiting_remote_revoke` = `false`
      - `message_awaiting_resp` = `false`
    - Process HTLCs that are impacted by the revocation:
      - The incoming HTLC is in `AwaitingAnnouncedRemoteRevoke`:
        - Promote to `Committed`
        - `require_commitment` = true
        - Add the HTLC to `pending_update_adds`
    - `monitor_pending_update_adds`: push htlcs here
    - `maybe_free_holding_cell_htlcs`: returns None
      - `require_commitment` is true:
        - `build_commitment_no_status_check`:
          - We don't have any HTLCs that need acting on
          - `resend_order` = `RevokeAndAckFirst` 
          - `awaiting_remote_revoke` = `true`
          - return a `ChannelMonitorUpdateStep::LatestCounterpartyCommitmentTxInfo`
        - `monitor_updating_paused(false, true, false, [], [] ,[])`
          - Updates state accordingly
    - We return updates:
      `CommitmentSecret` and `LatestCounterpartyCommitmentTxInfo`

```
Incoming Channel:
  pending_inbound_htlcs:
    state: InboundHTLCState::Committed

expecting_peer_commitment_signed: false
resend_order: RAACommitmentOrder::RevokeAndAckFirst
is_awaiting_remote_revoke: true
message_awaiting_resp: false

monitor_pending_revoke_and_ack = false
monitor_pending_commitment_signed = true
monitor_pending_forwards = []
monitor_pending_failures = []
monitor_pending_finalized_fulfills = []
monitor_monitor_update_in_progress = true
signer_pending_commitment_update = false

monitor_pending_update_adds = [UpdateAddHTLC msg]
```

- `handle_new_monitor_update` with our updates
  - `chainmonitor.update_channel` / `channelmonitor.update_channel` /
    `channelmonitor.inner.update_monitor`:
    - For `CommitSecret`
      - `ChannelMonitorImpl.provide_secret`
        - `commitment_secrets.provide_secret`: reports secret
    - For `LatestCounterpartyCommitmentTxInfo`:
      - `provide_latest_counterparty_commitment_tx`: 
        - Cache mapping of htlc payment hash to commit nr, for easy lookup
      - Update `channelmonitor`'s view of counterparty's current tx
        (update txid, number etc)
      - Store commitment points for counterparty 
   - `handle_monitor_updating_restored`:
     - Our `update_add_htlc` is in `monitor_pending_update_adds`
     - We are not `pending_revoke_and_ack`
     - We are waiting on a `pending_commitment_signed`:
       - `CommitmentUpdate` = `get_last_commitment_update_for_send`:
         - We don't have any `pending_outbound_htlcs`
         - We don't have any `LocalRemoved` `pending_inbound_htlcs`
         - `send_commitment_no_state_update`:
           - `build_commitment_transaction(local=false, gen by local=true)`
           - Sign commitment and return `CommitmentSigned` message
         - Assuming `signer_pending_commitment_update` = false,
           set `signer_pending_commitment_update` = true
     - Our `resend_order` is `RevokeAndAckFirst`, but we aren't
       `pending_revoke_and_ack` so we keep `Some(CommitmentUpdate)`
     - `pending_revoke_and_ack` = `false`
     - `pending_commitment_signed` = `false`
     - `MonitorRestoreUpdates` with:
       - `commitment_update` set
       - `pending_update_adds` contains our htlc

```
Incoming Channel:
  pending_inbound_htlcs:
    state: InboundHTLCState::Committed

expecting_peer_commitment_signed: false
resend_order: RAACommitmentOrder::RevokeAndAckFirst
is_awaiting_remote_revoke: true
message_awaiting_resp: false

monitor_pending_revoke_and_ack = false
monitor_pending_commitment_signed = false
monitor_pending_forwards = []
monitor_pending_failures = []
monitor_pending_finalized_fulfills = []
monitor_monitor_update_in_progress = true
signer_pending_commitment_update = true

monitor_pending_update_adds = []
```

- `(htlc_forwards, decode_update_add_htlcs) = handle_channel_resumption`
  - Push the `CommitmentUpdate` to `pending_msg_events`
  - We don't have a pending `raa`
  - We do not return any `hltc_forwards`, our `UpdateAddHTLC` is
    returned as `decode_update_add_htlcs`
  - We are using our _incoming_ channel's `outbound_scid_alias` and
    include the incoming `UpdateAddHTLC` in the vec
- `push_decode_update_add_htlcs`: pushes to `decode_update_add_htlcs`
  in `ChannelManager`

```
ChannelManager:
decode_update_add_htlcs: <incoming chan's outgoing alias, UpdateAddHTLC>
```
  - Decode htlc and check that we can forward it 
    `decode_update_add_htlc_onion`
  - Lookup peer and the channel the HTLC is for
  - `construct_pending_htlc_status` creates a `PendingHTLCStatus`:
    - `PendingHTLCStatus::Forward`
  - Check `can_accept_incoming_htlc` which looks at channel constraints
    (dust, in flight etc)
  - `try_chan_phase_entry` with `update_add_htlc`
    - Checks that the state of the channel and the HTLC are sane
      - Eg, we're not shutting down
      - Eg, the HTLC is non-zero / not larger than channel
    - Add to `pending_inbound_htlcs` if all checks are okay with state
      `RemoteAnnounced` / `Pending`

## Processing HTLC forwards

- `process_pending_htlc_forwards` is periodically called by a background
  processor:
  - `process_pending_update_add_htlcs`:
    - `should_persist` = `false`
    - Pull htlcs out of `decode_update_add_htlcs` an iter:
      - `should_persist` = `true`
      - We get all the data we need about the incoming channel
        with a `do_funded_channel_callback`
      - If the channel doesn't exist anymore, then we ignore the
        htlc, we should resolve it on chain
      - `decode_incoming_update_add_htlc_onion`:
        - `decode_next_payment_hop`: `Hop::Forward`
        - returns `(Hop::Forward, NextPacketDetails)`
      - Check `can_accept_incoming_htlc` on incoming channel:
        - Performs dust checks etc
      - Check `can_forward_htlc` on outgoing channel: 
        - `can_forward_htlc_to_outgoing_channel`:
          - Checks on values of the outgoing packet (eg amt, cltv)
        - `check_incoming_htlc_cltv`:
    - `get_pending_htlc_info`:
      - `create_fwd_pending_htlc_info`:
        - `RoutingInfo::Direct{amt_to_forward, outgoing_cltv_value}`
        - Push to `htlc_forwards`
    - Call `self.forward_htlcs(htlc_forwards)`:
      - Create `PendingAddHTLCInfo`
      - Push to `self.forward_htlcs`: `HTLCForwardInfo::AddHTLC(PendingAddHTLCInfo)`
    - Returns `true` because we added htlcs

```
Channel Manager:
  forward_htlcs: [HTLCForwardInfo::AddHTLC(PendingAddHTLCInfo)]

```
  - Pull the htlc out of our `forward_htlcs`
    - `process_forward_htlcs`:
      - We match on `HTLCForwardInfo::AddHTLC`
      - Create a `HTLCSource::PreviousHopData` with all of our
        `HTLCPreviousHopData`
      - `queue_add_htlc` on outgoing channel
        - `send_htlc(force_holding_cell = true)`:
          - Checks that we can add htlcs to the outgoing channel,
            balance availability and that the peer is online
          - Push a `HTLCUpdateAwaitingACK::AddHTLC` to `holding_cell_updates`
```
Channel Manager:
  forward_htlcs: []
   holding_cell_updates: [] 

```

Back in `process_pending_htlc_forwards`:
- `check_free_holding_cells`:
  - For each peer, for each channel:
    - `maybe_free_holding_cell_htlcs` / `free_holding_cell_htlcs`:
      - Pull HTLC out of `holding_cell_updates`
      - Match on`HTLCUpdateAwaitingACK::AddHTLC`
      - `send_htlc(force_holding_cell = false)`
        - Various state checks for the HTLC
        - `need_holding_cell` = false
        - Push a `OutboundHTLCOutput` to `pending_outbound_htlcs`
          with `OutboundHTLCState::LocalAnnounced(onion)`
```
Outgoing Channel:
pending_outbound_htlcs = [OutboundHTLCOutput(OutboundHTLCState::LocalAnnounced)]
```

In `free_holding_cell_htlcs`:
- `mon_update` = `build_commitment_no_status_check`
  - We don't have any `pending_inbound_htlcs`
  - We have an outbound htlc, but it's only in `LocalAnnounced`
  - `build_commitment_no_state_update`:
    - `build_commitment_transaction(local= false, gen by local= true)`:
      - Since we're on our commit, the `pending_outbound_htlc`
        is `included_in_commmitment`
      - We push it to `htlcs_included`
        - `LatestCounterpartyCommitmentTxInfo` with the htlc contained
           in `htlc_outputs`
  - `monitor_updating_paused(false, true, false, [], [], [])`
    - `monitor_pending_commitment_signed` = `true`
```
Outgoing Channel:
pending_outbound_htlcs = [OutboundHTLCOutput(OutboundHTLCState::LocalAnnounced)]

resend_order: RAACommitmentOrder::CommitmentSigned,
monitor_pending_commitment_signed = true
```

In `check_free_holding_cells`:
- `maybe_free_holding_cell_htlcs` has returned a 
  `LatestCounterpartyCommitmentTxInfo` update
- `handle_new_monitor_update` with `LatestCounterpartyCommitmentTxInfo`:
   - `chain_monitor.update_channel`
     - `monitor.update_monitor`
       - `provide_latest_counterparty_commitment_tx`:
         - Updates `prev_counterparty_commitment_txid` to last held
         - Sets `current_counterparty_commitment_txid` to new tx
     - Assuming `Completed`, no pending updates are returned
     - `provide_latest_holder_commitment_tx`
   - Assuming `ChannelMonitorUpdateStatus::Completed`:
     - `monitor_updating_restored`:
       - We don't have any pending HTLCs in our monitor now
       - `monitor_pending_revoke_and_ack` = false, so no `raa`
       - `monitor_pending_commitment_signed` = `true`
         - `get_last_commitment_update_for_send`
           - Our HTLC is `LocalAnnounced` in `pending_outbound_htlcs`:
             - Push `msgs::UpdateAddHTLC` to `update_add_htlcs` in our
               `CommitmentUpdate`
             - `send_commitment_no_state_update`:
               - `send_commitment_no_state_update_for_funding` 
                 - `build_commitment_transaction(local=false, gen_by_local=true)`
                - `sign_counterparty_commitment`
                  - `sign_counterparty_commitment` 
                  - Returns `Ok(CommitmentSigned)`
            - Returns`msgs::CommitmentUpdate` with `UpdateAddHTLC` and
              `CommitmentSigned`
       - `resend_order` is `CommitmentFirst`:
         - `signer_pending_revoke_and_ack` = `true`
      - `monitor_pending_revoke_and_ack` = `false`
      - `monitor_pending_commitment_signed` = `false`
```
Outgoing Channel:
pending_outbound_htlcs = [OutboundHTLCOutput(OutboundHTLCState::LocalAnnounced)]

resend_order: RAACommitmentOrder::CommitmentSigned,
monitor_pending_revoke_and_ack = false
monitor_pending_commitment_signed = false
```

We have `MonitorRestoreUpdates` with:
```
commitment_update = Some(CommitmentUpdate {
  UpdateAddHTLC, CommitmentSigned
})
```

- `handle_channel_resumption` with this `commitment_update`:
  - Our `commitment_order` is `CommitmentFirst`:
    - Push `UpdateHTLCs` message
    - We have no revoke and ack to send 
  - We don't have any signatures/broadcasting to do
  - There are no `hltc_forwards` or `decode_update_add_htlcs` to return
- `handle_monitor_update_completion_actions`: no pending actions
- No forwards/decodes/finalizes/fails to handle

## Receiving Revoke and Ack

Once we have send our peer a commitment_signed message, we expect to
receive a revoke and ack from them.

```
Outgoing Channel:
pending_outbound_htlcs = [OutboundHTLCOutput(OutboundHTLCState::LocalAnnounced)]

resend_order: RAACommitmentOrder::CommitmentSigned,
monitor_pending_revoke_and_ack = false
monitor_pending_commitment_signed = false
```

- `handle_revoke_and_ack`:
  - `internal_revoke_and_ack`:
    - Lookup channel in `peer_state.channel_id_by_entry`
    - `chan.revoke_and_ack`:
      - Validates the revocation message received
      - `ChannelMonitorUpdate` / `CommitmentSecret`
      - `is_awaiting_remote_revoke` = `false`
      - Sets commitments points for the remote peer
      - For each `pending_outbound_htlc`:
        - Upgrade `LocalAnnounced` to `OutboundHTLCState::Committed`
        - `expecting_peer_commitment_signed` = `true`
      - Assuming no `blocked_monitor_updates`
      - `maybe_free_holding_cell_htlcs` 
        - `free_holding_cell_htlcs`:
          - We don't have any HTLCs in our holding cells, so we return
            `None, []`
      - `require_commitment` is false
        - `monitor_updaing_paused(false, false, false, [], [], [])`
          - No changes to outgoing channel state since false & empty 
```
Outgoing Channel:
pending_outbound_htlcs = [OutboundHTLCOutput(OutboundHTLCState::Committed)]

resend_order: RAACommitmentOrder::CommitmentSigned,
monitor_pending_revoke_and_ack = false
monitor_pending_commitment_signed = false
expecting_peer_commitment_signed = true
```

- `handle_new_monitor_update` with `CommitmentSecret`
  - `update_channel`
    - `update_monitor`
      - `provide_secret`
  - Assuming `ChannelMonitorUpdateStatus::Completed`:
    - `updates` = `monitor_updating_restored`
      - `pending_revoke_and_ack` = `false`
      - `pending_commitment_signed` = `false`
      - Both `raa` and `commitment_update` are None, we don't owe
      - There also aren't any HTLCs we need to deal with
  - `handle_channel_resumption`: no actions
  - `handle_monitor_update_completion_actions`: no actions
  - `self.forward_htlcs(forwards)`: no actions

## Receiving Commitment Signed

We expect to receive a new commitment from our remote peer that adds
the HTLC to our commitment.

```
Outgoing Channel Context:
  resend_order: RAACommitmentOrder::RevokeAndAckFirst,
  monitor_pending_channel_ready: false,
  monitor_pending_revoke_and_ack: false,
  monitor_pending_commitment_signed: false,
  monitor_pending_forwards: [],
  monitor_pending_failures: [],
  monitor_pending_finalized_fulfills: [],
  monitor_pending_update_adds: [],
  is_awaiting_remote_revoke = false,
  pending_outbound_htlcs: OutboundHTLCOutput{ OutboundHTLCState::LocalAnnounced }
  holding_cell_updates: []

```

- `handle_commitment_signed`
  - `internal_commitment_signed`:
    - `chan.commitment_signed`
      - `funded_channel.commitment_signed`A
        - `validate_commitment_signed` 
          - `build_commitment_transaction` (`local=T`, `generated_by_local=F`)
            - Our HTLC is in `pending_outbound_htlcs` so it is included
            - Validate that the commitment and htlc sigs are ok
            - Returns `Ok(LatestHolderCommitmentTXInfo)`
        - `commitment_signed_update_monitor`
          - Our HTLC has not been settled/failed, so we don't change state
          - `need_commitment` = `F` (not set by any htlcs /fee updates)
          - `monitor_updating_paused(T, F, F, [], [], [])`

```
Outgoing Channel Context:
  resend_order: RAACommitmentOrder::CommitmentFirst,
  monitor_pending_channel_ready: false,
  monitor_pending_revoke_and_ack: true,
  monitor_pending_commitment_signed: false,
  monitor_pending_forwards: [],
  monitor_pending_failures: [],
  monitor_pending_finalized_fulfills: [],
  monitor_pending_update_adds: [],
  is_awaiting_remote_revoke = false,
  pending_outbound_htlcs: OutboundHTLCOutput{ OutboundHTLCState::LocalAnnounced }
  holding_cell_updates: []
```
   - `handle_new_monitor_update`
     - `update_channel`
       - `update_monitor`
         - `provide_latest_holder_commitment_tx`
           - Bumps our view of our current commitment transaction
    - Assuming `ChannelMonitorUpdateStatus::Completed`:
      - `updates` = `monitor_updating_restored`
        - No HTLCs that need handling
        - `monitor_pending_revoke_and_ack` = `T`
          - `get_last_revoke_and_ack`
        - `monitor_pending_commitment_signed` = `F`
        - `resend_order` = `CommitmentFirst` but `pending_commitment_update` = `F`
          so we don't wipe our our RAA
        - `pending_revoke_and_ack` = `F`
        - `pending_commitment_update` = `F`
     - `handle_channel_resumption`
       - Push `MessageSendEvent::SendRevokeAndAck`
     - There are no HTLC actions to take

```
Outgoing Channel Context:
  resend_order: RAACommitmentOrder::CommitmentFirst,
  monitor_pending_channel_ready: false,
  monitor_pending_revoke_and_ack: false,
  monitor_pending_commitment_signed: false,
  monitor_pending_forwards: [],
  monitor_pending_failures: [],
  monitor_pending_finalized_fulfills: [],
  monitor_pending_update_adds: [],
  is_awaiting_remote_revoke = false,
  pending_outbound_htlcs: OutboundHTLCOutput{ OutboundHTLCState::LocalAnnounced }
  holding_cell_updates: []
```

The HTLC is irrevocably committed on the outgoing channel!

## Update Fulfill HTLC

Once we're committed the HTLC, the downstream peer will send an
`update_fulfill_htlc` to settle the HTLC back.
