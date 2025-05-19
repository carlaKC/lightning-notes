# Commitment Building in LDK

Looking at this from two perspectives, opening up and updating a channel.

A: Channel Initiator
B: Channel Recipient

## A creates the channel

`create_channel`
- Lookup the peer in `per_peer_state`
  - Fail if not connected to the peer
- `channel` = `OutboundV1Channel`:
  - Generate a scid alias
  - Set config which contains details like:
    - Forwarding fees
    - `max_dust_exposure`
    - `accept_underpaying_htlcs`
  - `OutboundV1Channel::new`:
    - `new_for_outbound_channel`
- `channel.get_open_channel`:
  - Sanity checks that we can send an open channel message
  - Populates open channel with values from channel
- `peer_state.channel_by_id(temporary_channel_id)`
  - If occupied, the temp ID is bad so the RNG is bad
  - Push `Channel::from(channel)`
- Push `SendOpenChannel` to `peer_state.pending_msg_events`

```
peer_state.channel_by_id contains:
- temporary_channel_id: ChannelPhase::UnfundedOutboundV1(OutboundV1Channel)
```

Details:
```
OutboundV1Channel {

FundingScope:
  - value_to_self_msat
  - holder_selected_channel_reserve_satoshis
  - channel_transaction_parameters:
    - holder_pubkeys,
    - holder_selected_context_delay
    - channel_value_satoshis

ChannelContext:
  - channel_state: ChannelState::NegotiatingFunding {
        OUR_INIT_SET
    }
  - announcement_sigs_state: AnnouncementSigState::NotSent
  - resend_order: RAACommitmentOrder::CommitmentFirst 
  - All bools false to begin with

UnfundedChannelContext
  - unfunded_channel_age_ticks: 0
  - holder_commitment_point: HolderCommitmentPoint::new()
}
```

The `SendOpenChannel` event will be handled by A's `peer_handler` when
pending messages are cleared out, and B will receive an `open_channel`
message.

## B receives open_channel from A

`handle_open_channel`:
- `internal_open_channel`
  - Performs validation that the channel is valid + can be accepted
  - Limit total exposure to unconfirmed channels 
  - Get `per_peer_state` for pubkey
  - Error if we already have this channel ID tracked
  - `channel_type` = `channel_type_from_open_channel`
    - `ChannelTypeContext` defines the features that represent different
      types of channels.
    - If they provided a `channel_type` we use it
    - Otherwise it can be inferred from their `init`
  - LDK doesn't accept anchor channels if `manually_accept_inbound_channels`
    is false, because end users need to have reserve so they should make
    informed decisions about reserves.
    - See https://github.com/lightningdevkit/rust-lightning/issues/2320
    - So assume that it's true for the purposes of this walkthrough
  - `OpenChannelRequest` event pushed to pending events
  - `inbound_channel_request_by_id` = `InboundChannelRequest`
    - Keeps hold of the `open_channel` message received
    - Sets `ticks_remaining` to 2

To allow this channel to be opened, B's user will call `accept_inbound_channel`:
`do_accept_inbound_channel`:
- Lookup the channel in `inbound_channel_request_by_id`:
- Create a `InboundV1Channel::new`
  - `ChannelContext::new_for_inbound_channel`
- Call `accept_inbound_channel`:
  - Checks that it is appropriate to send the message
  - `generate_accept_channel_message` creates `AcceptChannel`
- Return a `SendAcceptChannel` event
- Push the message to `pending_msg_events`
- Insert the channel in `peer_state.channel_by_id`

```
peer_state.channel_by_id contains:
- temporary_channel_id: ChannelPhase::UnfundedInboundV1(InboundV1Channel)
```

The `SendAcceptChannel` will be handled by B's `peer_handler` when
pending messages are cleared out and A will receive an `accept_channel`
message.

## A receives accept_channel from B

The message will be handled by `handle_accept_channel`:
`internal_accept_channel`:
- Lookup channel in `peer_state.channel_by_id`:
  - `try_channel_entry!`: mostly handles error case
    - `unfunded_chan.accept_channel`:
      - `do_accept_channel_checks`:
        - Check that message can be received
        - Validates all the fields for the channel
        - Sets fields that can only be set from their response (eg, keys)
- Push a `FundingGenerationReady` message to `pending_events`

```
peer_state.channel_by.id contains:
- temporary_channel_id: ChannelPhase::UnfundedOutboundV1(OutboundV1Channel)
```

Details:
```
OutboundV1Channel {

FundingScope:
  - value_to_self_msat
  - holder_selected_channel_reserve_satoshis
  - channel_transaction_parameters:
    - holder_pubkeys,
    - holder_selected_context_delay
    - channel_value_satoshis

ChannelContext:
  - channel_state: ChannelState::NegotiatingFunding {
        OUR_INIT_SET | THEIR_INIT_SET
    }
  - announcement_sigs_state: AnnouncementSigState::NotSent
  - resend_order: RAACommitmentOrder::CommitmentFirst 
  - All bools false to begin with

UnfundedChannelContext
  - unfunded_channel_age_ticks: 0
  - holder_commitment_point: HolderCommitmentPoint::new()
}
```

The operator of A is responsible for calling `funding_transaction_generated`
with a `Transaction` for the channel.

`batch_funding_transaction_geneated`:
- `batch_funding_transaction_geneated_intern([temp_id, node_id], FundingType::Checked(tx)`
  - Validates all inputs are segwit
  - `is_manual_broadcast` is false for `Checked` 
  - `funding_transaction_generated_intern`:
    - Looks up the channel in `peer_state.channel_by_id` 
    - Calls the `find_funding_output` closure to get the txout
    - `get_funding_created`:
      - Asserts that the message can be sent
      - Sets funding outpoint for the channel
      - Sets channel state to `ChannelState::FundingNegotiated`
      - Checks minimum depth if it's a coinbase tx
      - Sets `channel_id` from the funding outpoint
    - Returns channel and message to push
  - Looks up the channel in `peer_state.channel_by_id`:
    - Checks that this channel ID isn't already being used
    - Pushes a `SendFundingCreated` message to `pending_msg_events`

```
peer_state.channel_by.id contains:
- channel_id: ChannelPhase::UnfundedOutboundV1(OutboundV1Channel)
```

Details:

```
OutboundV1Channel {

FundingScope:
  - value_to_self_msat
  - holder_selected_channel_reserve_satoshis
  - channel_transaction_parameters:
    - holder_pubkeys,
    - holder_selected_context_delay
    - channel_value_satoshis

ChannelContext:
  - channel_state: ChannelState::FundingNegotiated
  - announcement_sigs_state: AnnouncementSigState::NotSent
  - resend_order: RAACommitmentOrder::CommitmentFirst 
  - All bools false to begin with

UnfundedChannelContext
  - unfunded_channel_age_ticks: 0
  - holder_commitment_point: HolderCommitmentPoint::new()
}
```

The `SendFundingCreated` message will be handled by A's `peer_handler`
when pending messages are cleared out and B will receive a 
`funding_created` from A.

## B receives funding_crated from A

The message will be handled by `handle_funding_created`:
- `internal_funding_created`:
  - Lookup and remove the temporary channel in `peer_state.channel_by_id`:
    - `channel.funding_created`:
      - Validates that the message is appropriate for current state
      - `initial_commitment_signed`:
        - Checks that the counterparty's commitment is valid
        - `build_commitment_transaction`:
          - `build_commitment_stats`:
            - Calculates balances on either side and fees
            - Sets `total_anchors_sat` to 2x value
          - Collects outputs and HTLCs for the commitment
          - Subtracts anchors from appropriate balance
          - `CommitmentTransaction::new()`:
            - `build_outputs_and_htlcs`:
              - `build_sorted_htlc_outputs`:
                - `build_htlc_outputs` creates a vec of HTLCs
                - `build_sorted_htlc_outputs`: sorts them
              - Set the output index of each of the HTLCs
              - `insert_non_htlc_outputs` 
                - Calls provided closure for each output, creating
                  anchors and channel balances
                - Inserted in the correct order using the sort
            - `build_inputs` adds inputs to tx
            - `make_transaction`:
              - Sets version, locktime, inputs, outputs
            - Returns `CommitmentTransaction`
          - Returns `CommitmentData` with stats and tx
        - Create `HolderCommitmentTransaction`
        - Set `channel_state` to `ChannelState::AwaitingChannelReady`
        - Create a channel monitor and provide it with the first
          commitment transaction
      - `get_funding_signed_msg`:
        - Mark `counterparty_initial_commitment_tx` as `trust`ed, because
          we created it.
        - `sign_counterparty_commitment`:
          - Assuming using `InMemorySigner`:
            - `sign_counterparty_commitment`:
              - Takes `sighash_all`
              - Provides signature
            - Creates all the HTLC signatures 
        - `check_get_channel_ready`:
          - Checks that state is correct
          - Channel not confirmed so returns `None`
        - Return a `FundingSigned` message with the signature
      - Return a `FundedChannel` and `channel_monitor`
  - Lookup the real channel id in `channel_by_id`:
    - Fail if already populated
    - `chain_monitor.watch_channel`:
      - Add to `ChainMontior`'s `monitors` list
      - Persists new channel
      - `load_outputs_to_watch`
        - Registers outputs to watch on chain
    - Push a `SendFundingSigned`event to `pending_msg_events`
    - Insert `Channel::From(FundedChannel)`


```
peer_state.channel_by.id contains:
- channel_id: ChannelPhase::Funded(FundedChannel)
```

The `SendFundingSigned` message will be handled by B's `peer_handler`
when pending messages are cleared out and A will receive a 
`funding_signed` message from B.

## A receives a funding_signed message from B

This message will be handled by `handle_funding_signed`:
`internal_funding_signed`:
- Get the channel from `peer_state.channel_by_id`:
  - `channel.funding_signed`:

## Notes for V3

The `anchors_zero_fee_htlc_tx` is no longer aligned with the spec's
naming after https://github.com/lightning/bolts/pull/1092.

In `internal_open_channel` we'd also need to reject zero fee channels
by the same logic.

`build_commitment_stats` needs to have different anchor values.
`build_commitment_transaction`: do we still subtract anchor from funder?

`insert_non_htlc_outputs` needs to account for anchors
