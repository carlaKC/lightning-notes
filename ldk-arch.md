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
