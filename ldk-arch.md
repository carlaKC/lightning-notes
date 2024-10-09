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

## ChannelManager

- Has some node-level items, like pending payments and receives
- Forwards are expressed generically, not per-channel
- Pending channels are tracked in a single batch
- There is some per-peer state tracked:
  - `channel_by_id`: All channels with the peer by chan_id, holds
    the full channel state if the channel has been fully funded
  - Messages to send to the peer 

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
    - `create_fwd_pending_htlc_info`: Forward the HTLC
    - `create_recv_pending_htlc_info`: Receive the payment
    - Otherwise error accordingly
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
    - `build_commitment_transaction`:
      - Gets inbound `pending_inbound_htlcs` and `pending_outbound_htlcs`
      - Get the latest fee rate for the channel 
      - For each pending inbound htlc, based on state `add_htlc_output`:
        - HTLC is `RemoteAnnounced` so we will add it
      - For each pending outbound htlc, based on state `add_htlc_output`:
        - We have none, as we've only got one pending inbound 
    - Verifies signature on expected commitment transaction
    - Ensure that our counterparty can afford the new fee/commitment/
      reserve
    - Next build all HTLCs and validate their signatures
    - If we have a pending update fee from the remote party:
      - Note that they need a new commitment from us
      - Update the fee update's state to awaiting revocation
    - For each `pending_inbound_htlcs`:
      - Update state to `AwaitingRemoteRevokeToAnnounce`
    - We don't have any pending outgoing HTLCs at this point, so there's
      no action for `pending_outbound_htlcs`
  - Create a `ChannelMonitorUpdate` that has the new commitment info
  - `monitor_updating_paused`:
    - Called when we know that we haven't persisted an update for the
      `ChanelMonitor`
    - There are a bunch of state-machine related varaibles here:
      - pending_revoke_and_ack / commitment_signed etc
  - Update is pushed onto the channel's queue:
    - If there are blocked updates, the user can't handle it now
    - If there's nothing queued, then return for the user to persist
    - These are stored in `blocked_monitor_updates` on the channel
  - `handle_new_monitor_update`
    - update_channel: notify chain monitor of new updates
    - `handle_monitor_update(self, updates_res, chan, _internal)`:
      - ??
Resume on 8315
handle_new_monitor_update(
  ident: ChannelManager,
  expr: FundingUTXO,
    ChannelMonitorUpdate {
	  update_id: self.context.latest_monitor_update_id,
	  counterparty_node_id: Some(self.context.counterparty_node_id),
	  updates: vec![ChannelMonitorUpdateStep::LatestHolderCommitmentTXInfo {
	    commitment_tx: holder_commitment_tx,
	    htlc_outputs: htlcs_and_sigs,
	    claimed_htlcs,
	    nondust_htlc_sources,
	  }],
	channel_id: Some(self.context.channel_id()),
  },
  expr: MonitorUpdate,
  expr: peer_state_lock,
  expr: peer_state,
  expr: per_peer_state, 
  expr: Channel,
)

### Handling Update

### Sending Revoke and ACK

What background task triggers a revoke and ack?
- Probably done in `handle_new_monitor_update`?
handle_monitor_update_completion calls handle_channel_resumption

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
