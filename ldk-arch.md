# High Level Arch

ChannelManager:
- Keeps track of node's channels
- Forwards HTLCs
- Sends/Receives payments

ChannelMonitor:
- Tracks on-chain state of a channel
- Manages justice and force closes 

P2PGossipSync:
- Handles channel/node announcements
- Provides pathfinding for `find_route`

PeerManager:
- Handles P2P messages
  - Routes to ChannelManager
  - Routes to P2PGossipSync

Must be provided by user:
- Key generation
- Tx broadcast
- Block fetching
- Fee estimation

The library does not depend on a runtime, so some functions must be 
called on a timer - only the calling code drives creation of new tasks.

## Interactions

P2PGossipSync -> UtxoLookup:
- Use the `get_utxo` function to check that channels exist when we 
  validate `channel_announcement`

PeerManager -> P2PGossipSync (as RoutingMessageHandler):
- PeerManager is provided with a set of APIs to report any gossip 
  related messages to
- Also has lookup functions which can be used for pathfinding

PeerManager -> ChannelManager (as ChannelMessageHandler):
- PeerManager is provided with a set of APIs to report any channel
  related messages to the ChannelManager
- Also has some "get" methods to understand the features/chain that the
  node supports

ChannelManager calls out to a bunch of hooks!

Q: docs/names seem wrong here?
ChannelManager -> MessageSendEvent (as MessageSendEventsProvider):
- get_and_clear_pending_msg_events: pulls the messages from the 
  ChannelManager to be delivered to peers

ChannelManager -> UserConfig:
- Allows setting various user-tweakable settings, such as whether to
  accept inbound channels, enable HTLC interception etc.

ChannelManager -> FeeEstimator:
- `get_est_sat_per_1000_weight` provides current fee estimation

ChannelManager -> BroadcasterInterface:
- `broadcast_transactions` sends a list of transactions to the bitcoin
  network for mining, LDK will handle rebroadcast.

ChannelManager -> chain::Watch:
- Provides a set of APIs to inform the chain watcher about current 
  channels that must be watched.
- Poll watcher for any relevant on-chain activity (such as a force 
  close, a funding event etc) with `release_pending_monitor_events`
  - Implemented by ChainMonitor

ChannelManager -> MessageSendEvent (as MessageSendEventsProvider) -> PeerManager
- ChannelManager implements `get_and_clear_pending_msg_events` which 
  provides a vector of MessageSendEvent items which provide the 
  messages that need to be sent out to peers
- The peer manager periodically polls this to fetch messages that need 
  to be dispatched

ChannelManager -> KeysInterface:
- This interface has since been broken up into three parts:
  - EntropySource: generates secure signing bytes
  - SignerProvider: provides signing operations for an individual channel.
  - NodeSigner: provides node-specific signing (ecdh and invoices)

ChainMonitor -> FeeEstimator:
- ChainMonitor uses a fee estimator to handle rebroadcasts, and get 
  fee rates for on-chain claims that are triggered by new blocks arriving

ChainMonitor -> chain::Filter:
- ChainMonitor calls register_tx and register_output on Filter to tell 
  the underlying chainsource which transactions it's interested in.
  - Useful, for example, for compact blocks so that we don't have to
    fetch each thing

ChannelManager (as EventsProvider) -> Event:
- process_pending_events is passed an event handler closure which 
  should handle omitted events accordingly

ChainMonitor (as EventsProvider) -> Event:
- process_pending_events is passed an event handler closure which 
  should handle omitted events accordingly

## Background Processor
- Runs all tasks that are required to keep LDK happy:
  - Need to happen periodically for regular operation
  - Can/should run in the background
- Processes Events, passing them to user-defined handlers
- Managing persistence of `ChannelManager`
- Driving tick-based APIs on various structs 
- Handling stale gossip, if configured
- Runs until:
  - Struct is dropped
  - Persistence fails
- Created with various user-provided hooks:
  - Persister: can write channel manager, graph and scorer
  - EventHandler: implements `handle_event` to manage the events that
    LDK outputs
  - Plus the three key LDK structs: ChannelManger, PeerManager, ChainMonitor
  - OnionMessenger: handles onion messages
  - Optional gossip stuff
  - Logger

Event Handling Closure:
- Handle events that need to update scorer or network graph:
  - If they're provided, call functions that match to relevant events,
    otherwise no-op
- Finally, call the user-provided closure with the event

# Details

## ChannelMonitor

- `ChannelMonitorUpdate` represents an update to a channel
  - `ChannelMonitorUpdateStep` has the actual update:
    - `LatestHolderCommitmentTXInfo`: we have received a commitment
      signed from our counterparty, has all sigs, htlcs, preimages
      and forwards that we need to remember
    - `LatestCounterpartyCommitmentTXInfo`: we have sent a commitment
      signed to our counterparty, has all htlcs, balances and commitment
      points required.
    - `PaymentPreimage`: we've received a preimage
    - `CommitmentSecret`: we've received a secret
    - `ChannelForceClosed`: terminal signal for channel
    - `ShutdownScript`: provides address for shutdown
  - Stores pubkey so that `ChannelMonitor` can look up the channel 
    easily 
  - `update_id`: sequence id, must be replayed in order
- [Possibly wrong thought]: while the channel monitor is mostly 
  concerned with handling on-chain transactions, its updates actually
  holding a lot of the information that we'll use in the ChannelManger,
  that's why we need to pull the `ChannelMonitor` from disk to start
  up the `ChannelManger`.
  - As an example,  

`ChannelMonitorUpdateStatus`:
- Used to represent the status of an update to the channel monitor's
  persistence. 
- `InProgress` is the interesting state (otherwise done or failed),
  which is used to freeze the channel (indicating that persistence is
  happening asynchronously or a retryable error has occurred).

## ChannelManager

- Has some node-level items, like pending payments and receives
- Forwards are expressed generically, not per-channel
- Pending channels are tracked in a single batch
- There is some per-peer state tracked:
  - `channel_by_id`: All channels with the peer by chan_id, holds
    the full channel state if the channel has been fully funded
  - Messages to send to the peer 

Node Payments:
- `pending_inbound_payments`: stores preimages + secrets for our payments
  that we're waiting to report to the user as `PaymentClaimable`
- `pending_outbout_payments`: session information and state that we
  need for our own payments
- `claimable_payments`: payments that can currently be claimed

Node HTLCs:
- `forward_htlcs`: HTLCs that need to be forwarded  
  - SCID = 0 means that we've received a payment
- `pending_intercepted_htlcs`: holds intercepted HTLCs that are waiting
  from an action from the user
- `decode_update_add_htlcs`: what's the difference here 

Node
- `per_peer_state`:
  - `channel_by_id`
    - `ChannelPhase`:
      - Enum for each possible channel state
      - `Funded(Channel)` for confirmed channels:
        - `ChannelContext` holds channel's state
  - `pending_msg_events`: messages that must be sent to the peer
- `pending_events`: events that need to be handed to the user

Restarts:
- To reload from disk, the `ChannelManager` requires all of the
  deserialized `ChanelMonitor`s that we stored
- `ChannelManager` is loaded from disk + args provided elsewhere,
  if there are inconsistencies with `ChannelMonitor` you may trigger a
  force close

Writable: what does `ChannelManager` actually persist?
- Chain hash, best block height and hash
- Channel count + each `Channel`:
  - Channel ID
  - Channel state (in funding process)
  - Channel size sats
  - latest monitor update id <- what's this?
  - Upfront shutdown pubkey
  - Current commitment number for each party
  - Any HTLCs that we need to remember (inbound and outbound)
  - Preimages for HTLCs that we know
  - Whether to send RAA of Commitment next / what's pending in exchange
  *TODO*: get full picture of what's stored here
- HTLC count:
  - SCID
    - Forward count: forwards:
      - Adds: PendingHTLCInfo
      - Failures: HTLC ID + ErrPacket
      - Malformed: HTLC ID + code + dummy err pkt + sha
- Claimable payments count / payments:
  - Hash / payment HTLCs:
    - what's written for HTLCs

TODO: continue looking at what's persisted (L11894)


### Commitment Dance

Full commitment dance for adding a HTLC that's intended to be forwarded.

Starting state for channel:
- When we receive a `channel_ready` and we've sent our own, we'll
  progress our `ChannelState` enum on `Channel` to `ChannelReady`
  - Pull all flags from previous state and set `FundedStateFlags::ALL`

#### Adding a HTLC
- Once a channel has been fully opened, it's going to be tracked in
  the `ChannelManger`'s `per_peer_state`:
  - `channel_by_id` will have `ChannelPhase::Funded(Channel)`
- When the `PeerManager` gets an `update_add_htlc`, it'll call 
  'handle_update_add_htlc` on its `ChannelMessageHandler` (which is
  implemented by `ChannelManger`
- `internal_update_add_htlc` returns an indication of whether we need
  to do any persistence after this operation:
  - If something has gone so wrong that we have to close the channel,
    then we notify that persistence is required
  - SkipPersistHandleEvents:
  - SkipPersistNoEvents:
- `internal_update_add_htlc`:
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
      `RemoteAnnounced`

### Receiving a Commitment
- When the `PeerManager` gets a `commitment_signed`, it will call
  `handle_commitment_signed` on its `ChannelMessageHandler` (which is
  implemented by `ChannelManager`)
- We `notify_on_drop` the `PersistenceNotifier`, indicating that we 
  must persist once this function is done
- `internal_commitment_signed`:
  - `local` = true (determines direction of HTLCs for tx)
  - `generated_by_local` = false (only add proposed htlcs)
  - Fetches the channel and asserts that it's reached the funded stage
  - `try_chan_phase_entry` with `commitment_signed`
    - Sanity checks on the channel's state for update
    - `build_commitment_transaction` (generated_by_local = F)
      - For each pending inbound htlc, based on state `add_htlc_output`:
        - HTLC is `RemoteAnnounced` so we will add it to commitment
    - For each `pending_inbound_htlcs`:
      - Update state to `AwaitingRemoteRevokeToAnnounce`
    - We don't have any pending outgoing HTLCs at this point, so there's
      no action for `pending_outbound_htlcs`
  - Create a `ChannelMonitorUpdate` that has the new commitment info
    - `LatestHolderCommitmentTXInfo`: holds sigs and commit info
  - `expecting_peer_commitment_signed` = F 
  - `resend_order` = `CommitmentFirst
  - `is_monitor_update_in_progress` = false (don't branch)
  - `need_commitment` is true (because we have htlcs) and 
    `is_awaiting_remote_revoke` is false, so we'll call
    `build_commitment_no_status_check`:
    - Upgrades HTLCs from `AwaitingRemoteRevokeToAnnounce` to 
      `AwaitingAnnouncedRemoteRevoke`
    - `resend_order` = RevokeAndAckFirst
    - `awaiting_remote_revoke` = T 
    - Append update to `monitor_update`:
      - LatestCounterpartyCommitmentTXInfo 
    - `need_commitment_signed` = T
    - `awaiting_remote_revoke` = T
  - `monitor_updating_paused`(true, true, false, {empty vecs}:
    - Called when we know that we haven't persisted an update for the
      `ChanelMonitor`
    - There are a bunch of state-machine related variables here:
      - `pending_revoke_and_ack` = true
      - `pending_commitment_signed` = true
	  - `pending_channel_ready` = false
      - `pending_forwards/failures/finalized` = empty
	  - `update_in_progress` = T
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
            `CommitmentSigned` it does not have any HLTCs on it because 
            the monitor's various `pending_forwards/failures/updates_adds`
            are not yet set
      - Call `handle_channel_resumption`:
        - Push `SendRevokeAndAck` message
        - Push `SendCommitmentSigned`
        - There aren't any HTLCs to handle at this stage

### Receiving Revoke and ACK

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
        - Add it to to_forward_infos
    - Assuming `mointor_update_in_progress` is false and
      `free_holding_cell_htlcs` are empty:
      - `monitor_updaing_paused`:
        - Pushes HTLC to `monitor_pending_forwards`
        - `mointor_udpate_in_progress` = T
        - `pending_revoke_and_ack` = F (unchanged)
        - `pending_commitment_signed` = F (unchanged)
      - `return_with_htlcs_to_fail`:
        - `release_monitor` = F && !false = F
        - Just returns `monitor_update`
  - `handle_new_monitor_update`
      - `monitor_update`: CommitmentSecret
      - `chain_monitor.update_channel`:
        - `update_monitor`:
          - `provide_secret`
      - updates = `monitor_updating_restored`
        - There are no pending commitment/raa in the mointor
        - returns `accepted_htlcs` including our forward 
      - `htlc_forwards, decode_update_adds` =`handle_channel_resumption`
        - Returns HTLC in `htlc_forwards`
        - No messages/events need handling
      - `handle_monitor_update_completion_actions`: TODO!
      - if `htlc_forwards` ->`forward_htlcs`
        - Pushes HTLC to the forward_htlcs mutex map

## Channel

- `define_state_flags!` macro (using `ChannelReady` example):
  - Document for the flag 
  - FUNDED_STATE adds extra flage to our definition
  - Name of the flag being implemented (eg `ChannelReadyFlags`)
  - List of flags that we're adding:
    - Doc string for the flag
    - The name of the flag
    - The bit value of the flag that we set
    - Name of the getter function to generate
    - Name of the setter function to generate
    - Name of the function to clear the flag

- `impl_state_flag!` macro:
  - Implements methods on `ChannelState` that provide getter/setter/clear
    for a given ChannelState
  - Matches current state to the expected state, then calls the method
    on the flags
  - Each variant of the `ChannelState` enum holds its own set of flags
    that are defined by the `define_state_flags` macro

## ChannelMonitor

- Responsible for the on-chain actions related to a channel
- We get a channel monitor back in the funding dance when we need to 
  start watching the chain (ie, we've signed)
- We call `ChainMonitor.watch_channel` to report the channel's existence
- Channel mointors are given to the `ChannelManger` via 
  `ChannelManagerReadArgs`: things that we don't

## STFU Implementation in LDK

`msgs.rs`:
- Contains all of the p2p messages in the LN protocol
- Derives a pretty standard set of traits (Clone/Debug/PartialEq/Eq)
- There is a `impl_writable_msg` macro which implements 
  Readable/Writable traits for most of the messages in LDK (some have
  custom):
  - Provide message name
  - Provide list of fields that are written "normally" (ie, non-TLV)
  - Provide a list of TLV numbers, the corresponding field and optional)
- The Encode trait's TYPE const is used to match message type to struct

ChannelMessageHandler:
- Trait that describes an object that can receive channel-specific
  messages includes a `handle_stfu` message that has `their_node_id` and
  the `STFU` message.
- Extends MessageSendEventsProvider, which just contains get_and_clear
  messages fn
- This is called when we receive a stfu messsage in PeerManager
-> Not yet implemented under the hood

Possible implementation:
- We can just look at the HTLC state of our incoming and outgoing HTLCs
  to determine whether we're able to STFU

# To Further Investigate

- HTLC Interception
