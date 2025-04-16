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
  - Performs various state/value sanity checks
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

## Receiving a Commitment
- When the `PeerManager` gets a `commitment_signed`, it will call
  `handle_commitment_signed` on its `ChannelMessageHandler` (which is
  implemented by `ChannelManager`)
- We `notify_on_drop` the `PersistenceNotifier`, indicating that we 
  must persist once this function is done
- `internal_commitment_signed`:
  - Fetches the channel and asserts that it's reached the funded stage
  - `try_channel_entry` with `commitment_signed`
    - Sanity checks on the channel's state that an update is possible
    - `build_commitment_transaction`:
      - `local=true`
      - `generated_by_local=false`
      - HTLC is in `pending_inbound_htlcs` with state `RemoteAnnounced`
        so we'll call `add_htlc_output`:
        - `outbound=false`
        - Added to `included_non_dust_htlcs`: 
        - Included in the commitment transaction
    - Validates all expected signatures
    - Creates a `HolderCommitmentTransaction` with sigs + commit
    - For each `pending_inbound_htlcs`:
      - Update state to `AwaitingRemoteRevokeToAnnounce`
      - `need_commitment = T`
    - We don't have any pending outgoing HTLCs at this point, so there's
      no action for `pending_outbound_htlcs`
  - Create a `ChannelMonitorUpdate` that has the new commitment info
    - `LatestHolderCommitmentTXInfo`: holds sigs and commit info
  - Updates to `self.context`:
    - `expecting_peer_commitment_signed` = F 
    - `resend_order` = `CommitmentFirst
    - `is_monitor_update_in_progress` = false (don't branch)
  - Assuming that `!is_monitor_update_in_progress`
  - `need_commitment` && `!is_awaiting_remote_revoke`:
    - `build_commitment_no_status_check`:
      - Upgrades HTLCs from `AwaitingRemoteRevokeToAnnounce` to 
        `AwaitingAnnouncedRemoteRevoke`
      - `self.resend_order` = RevokeAndAckFirst
      - `self.awaiting_remote_revoke` = T 
    - Returns a `LatestCounterpartyCommitmentTXInfo`
  - Append update to `monitor_update`
  - `monitor_updating_paused`(true, true, false, {empty vecs}:
    - Called when we know that we haven't persisted an update for the
      `ChanelMonitor`
    - There are a bunch of state-machine related variables here:
      - `self.pending_revoke_and_ack` = true
      - `self.pending_commitment_signed` = true
	  - `self.pending_channel_ready` = false
      - `self.pending_forwards/failures/finalized` = empty
	  - `self.update_in_progress` = T
      - `self.monitor_update_in_progress` = T
  - Update is pushed onto the channel's queue:
    - If there are blocked updates, the user can't handle it now
    - If there's nothing queued, then return for the user to persist
    - These are stored in `blocked_monitor_updates` on the channel
  - `handle_monitor_update`
    - Gets `in_flight_monitor_updates`
    - Call `chain_monitor.update_channel`
      - Gets the specific `chain_monitor` and acquires its lock:
        - `channelmonitor.update_monitor`:
          - For each update in the queue:
            - `LatesttHolderCommitmentTxInfo`
              - `provide_latest_holder_commitment_tx`:
              - Reports our latest tx to the `onchain_tx_handler`
              - Updates its view of the current commitment tx
            - `LatestCounterpartyCommitmentTXInfo`
              - `provide_latest_counterparty_commitment_tx`
              - Likewise notifies the chain monitor of new tx
      - If there are no updates pending on channel monitor:
        - `monitor_updating_restored` returns updates:
          - `pending_revoke_and_ack` = T; create `revoke_and_ack`
            - `signer_pending_revoke_and_ack=false`
          - Update state machine:
          - `pending_revoke_and_ack` = F
          - `pending_commitment_signed` = F
          - Returns a set of `MonitorRestoreUpdates` with a `RAA` and
            `CommitmentSigned` it does not have any HTLCs on it because 
            the monitor's various `pending_forwards/failures/updates_adds`
            are not yet set
      - Call `handle_channel_resumption`:
        - Push `SendRevokeAndAck` message
        - Push `SendCommitmentSigned`
        - There aren't any HTLCs to handle at this stage

## Receiving Revoke and ACK

- When the `PeerManager` gets a `revoke_and_ack`, it will call 
  `handle_revoke_and_ack` on its `ChannelMessageHandler` (which is
  implemented by `ChannelManager`
- We `notify_on_drop` the `PersistenceNotifier`, indicating that we
  must persist once this function is done
- `internal_revoke_and_ack`:
  - Looks up peer/channel as above, and asserts that the channel is in
    the correct state
  - `try_chan_phase_entry` with `revoke_and_ack` 
    - Check that revocation point is as expected for previous commit
    - Create `ChannelMonitorUpdate` with a `CommitmentSecret` update
    - Clear state on `Channel`:
      - `awaiting_remote_revoke` = F
      - `message_awaiting_resp` = F
    - Process HTLCs that are impacted by the revocation:
      - The incoming HTLC is in `AwaitingAnnouncedRemoteRevoke`:
        - Promote to `Committed`
        - `require_commitment` = true
    - Assuming `mointor_update_in_progress` is false and
      `free_holding_cell_htlcs` are empty:
      - `require_commitment` = T
      - `monitor_updaing_paused`:
        - `pending_revoke_and_ack` = unchanged
        - `pending_commitment_signed` = T (set)
        - `pending_channel_ready` = unchanged
        - `monitor.pending_forwards` = HTLC
        - `monitor.update_in_progress` = T (set)
      - `return_with_htlcs_to_fail`:
        - `release_monitor` = F && !false = F
        - Just returns `monitor_update` (`CommitmentSecret`)
  - `handle_new_monitor_update`
      - `monitor_update`: CommitmentSecret
      - `chain_monitor.update_channel`:
        - `update_monitor`:
          - `provide_secret`
      - updates = `monitor_updating_restored`:
        - There are no pending commitment/raa in the monitor
        - HTLC is included in `updates.accepted_htlcs` in 
          `MonitorRestoreUpdates`
      - `handle_channel_resumption`:
        - `pending_forwards` contains our htlc, returned as `htlc_forwards`,
          which is a vec of `PendingHtlcInfo` and some channel info
      - `handle_monitor_update_completion_actions`:
        - No actions to take here
      - `self.forward_htlcs(forwards)`

## Forwarding a HTLC

```
Channel Manager:
  forward_htlcs: []

```

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

```

- Once the `ChannelManager` has locked in the incoming htlc, it'll call
  `self.forward_htlcs(htlc)` with the incoming htlc.
  - `push_forward_event` = `forward_htlcs_without_forward_event`
    - push_forward_event = false
    - HTLC is passed in `pending_forwards`
    - For each pending_forwards (draining vec):
      - SCID is non-zero because it's a forward 
      - The forward's scid is not yet in `self.forward_htlcs`,
        and `is_our_scid` is true
        - `forward_htlcs_empty` && `decode_update_add_htlcs_empty`
          are both true (we don't have anything yet)
        - We push `HTLCForwardInfo::AddHTLC(PendingAddHTLCInfo)` to
          `self.forward_htlcs`
  - Returns with `push_forward_event=true`
  - `push_pending_forwards_ev`:
    - Pushes a `PendingHTLCsForwardable` event to `pending_events`

## Forwarding Pending HTLCs

```
Channel Manager:
  forward_htlcs: [scid] [HTLCForwardInfo::AddHTLC(PendingAddHTLCInfo)]

```

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

```

- Pushing a `PendingHTLCsForwardable`, which means that the end user
  needs to call `process_pending_htlc_forwards`
  - `process_pending_update_add_htlcs`:
    - no `decode_update_add_htlcs`, NOP
  - `forward_htlcs` contains our `AddHtlc::PendingAddHtlcInfo`, we
    iterate through each channel's HTLCs to forward (this is a vec,
    there can be many):
    - Lookup the fowarding counterparty and outgoing channel id
    - Drain all `pending_forwards` for the channel:
      - Our HTLC is a `HTLCForwardInfo::AddHTLC`
      - Pick optimal channel (non-strict forwarding)
      - `queue_add_htlc` with `HTLCSource::PreviousHopData`
        -  `send_htlc` (`force_holding_cell`= T):
          - Make sure that the channel is ready / online / not shutting down
          - Perform various checks that the HTLC is sane (amount/policy)
          - Push `HTLCUpdateAwaitingACK::AddHTLC` into
            `holding_cell_htlc_updates`
          - Return `Ok(None)`
- `pending_outbound_payments.check_retry` nop because we don't have payments
- `failed_forwards` is empty because we added our htlc successfully

```
Channel Manager:
  forward_htlcs: []

```

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

  holding_cell_updates: HTLCUpdateAwaitingACK::AddHTLC

```

- `check_free_holding_cells`:
  - Loops through each of our channels in `per_peer_state`
    - `maybe_free_holding_cell_htlcs`
      - `free_holding_cell_htlcs`:
        - Create `ChannelMonitorUpdate`
        - `holding_cell_htlc_updates` has our HTLC in it
        - `HTLCUpdateAwaitingACK::AddHTLC`:
          - `send_htlc` (`force_holding_cell` = F)
            - This time `force_holding_cell` is false
            - `need_holding_cell` = !`can_generate_new_commitment` = !`T`:
              - We're not shutting down / quiescing
              - `awaiting_remote_revoke` = `F`
              - = `F`
            - `pending_outbound_htlcs` push
              - `OutboundHTLCOutput`
              - `OutboundHTLCState::LocalAnnounced`
            - Return `Some(UpdateAddHTLC)`
          - Set `update_add_count` +1
    - `update_add_count` != 0 so we proceed with commitment building
    - `build_commitment_no_status_check`:
      - We have `pending_outbound_htlcs`, but state is `LocalAnnounced` 
      - `resend_order` = `RevokeAndAckFirst` 
      - `build_commitment_no_state_update`:
        - `build_commitment_transaction` (`local=F`, `gen_by_local=T`)
          - Iterate through `pending_outbound_htlcs`:
            - `htlc.state.included_in_commmitment(T)` = `T`
            - `add_htlc_output` will push to `included_non_dust_htlcs`
        - Returns our HTLC + a commitment tx
      - Push `ChannelMonitorUpdateStep::LatestCounterpartyCommitmentTXInfo`
        which includes our HTLC in its htlc outputs (local = F)
  - `mointor_updating_pasued(F,T,F, [],[],[])`
      - `monitor_pending_revoke_and_ack |= F` -> `F` (unchanged)
	  - `monitor_pending_commitment_signed |= T` -> T (changed)
	  - `monitor_pending_channel_ready |= F` -> (unchanged)
    - `push_ret_blockable_mon_update`:
      - Assuming that `blocked_monitor_updates.is_empty()`
      - `Some(updates)` 

```
Outgoing Channel Context:
  resend_order: RAACommitmentOrder::RevokeAndAckFirst,
  monitor_pending_channel_ready: false,
  monitor_pending_revoke_and_ack: false,
  monitor_pending_commitment_signed: true,
  monitor_pending_forwards: [],
  monitor_pending_failures: [],
  monitor_pending_finalized_fulfills: [],
  monitor_pending_update_adds: [],

  pending_outbound_htlcs: OutboundHTLCOutput{ OutboundHTLCState::LocalAnnounced }
  holding_cell_updates: []

```
   - `handle_new_monitor_update`  (`LatestCounterpartyCommitmentTXInfo`)
     - `chain_monitor.update_channel`
       - `monitor.update_monitor`
         - `provide_latest_counterparty_commitment_tx`:
           - Updates `prev_counterparty_commitment_txid` to last held
           - Sets `current_counterparty_commitment_txid` to new tx
       - Assuming `Completed`, no pending updates are returned
       - `provide_latest_holder_commitment_tx`
     - Assuming `ChannelMonitorUpdateStatus::Completed`:
       - `monitor_updating_restored`
         - We don't have any pending HTLCs in our monitor now
         - raa = None (`monitor_pending_revoke_and_ack` = F)
         - `monitor_pending_commitment_signed` = T
           - `get_last_commitment_update_for_send`
             - Our HTLC is `LocalAnnounced` in `pending_outbound_htlcs`:
               - Push `msgs::UpdateAddHTLC` to `update_add_htlcs`
               - `send_commitment_no_state_update`:
                 - `send_commitment_no_state_update_for_funding` 
                   - `build_commitment_transaction` (`local=F`, `gen_by_local=T`)
                     - Does the same as above
                  - `sign_counterparty_commitment`
                    - `sign_counterparty_commitment` 
                    - Returns `Ok(CommitmentSigned)`
              - Returns`msgs::CommitmentUpdate`
         - `resend_order` = `RevokeAndAckFirst` but 
           `signer_pending_revoke_and_ack` is false, so we do not
           overwrite `raa` or `commitment_update`
        - `monitor_pending_revoke_and_ack` = `F`
        - `monitor_pending_commitment_signed` = `F`
        - `MonitorRestoreUpdates` with `commitment_update`:
          - `update_add_htlcs` has our HTLC in it
          - `commitment_signed`: has our newly signed commitment tx
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

  pending_outbound_htlcs: OutboundHTLCOutput{ OutboundHTLCState::LocalAnnounced }
  holding_cell_updates: []

```

   - `handle_channel_resumption`
     `commitment_update`
     - Push a `MessageSendEvent::UpdateHTLCs` message 
       to `peer_state.pending_msg_events`
   - `handle_monitor_update_completion_actions`
     - `forward_htlcs`: no HTLCs to forward
     - `push_decode_update_add_htlcs`: no pending to decode
     - `finalize_claims`: no claims to finalize
   - `new_events` is empty (it's only filed when we can claim a payment)


## Receiving Revoke and Ack

Once we have send our peer a commitment_signed message, we expect to
receive a revoke and ack from them.

```
Channel Manager:
  forward_htlcs: []
```

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

  pending_outbound_htlcs: OutboundHTLCOutput{ OutboundHTLCState::LocalAnnounced }
  holding_cell_updates: []

```
