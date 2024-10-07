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

# STFU Implementation in LDK

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

# To Further Investigate

- HTLC Interception
