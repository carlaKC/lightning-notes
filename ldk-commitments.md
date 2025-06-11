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

TODO: finish channel funding flow

# Update Fee

Once a channel is established, how do we go about updating fees?

## Local Update

We'll call `ChannelManager.timer_tick_occurred` periodically to run
state updates that need to happen once a minute or so. This runs
for each `per_peer_state` and each `channel_by_id`.

For every `as_funded_mut` channel (ie, once that's funded):
- We get two possible fee rates:
  - Non anchor channel fee: for channels without anchors
  - Anchor channel fee: for channels with anchors
- `update_channel_fee`:
  - If the feerate has halved, don't worry (we don't want to bump fee down)
  - `queue_update_fee`
    - `send_update_fee`
      - Pushes into holding cell because force_holding_cell is true
- If this returns a `DoPersist` we know we need to write this channel.

Dispatching update fee:
- `fee_holding_cell_htlcs`
  - Grabs update from `holding_cell_update_fee`
    - `send_uddate_fee` with `force_holding_cell=false`
      - This actually returns the `UpdateFee` message we should send

## Remote Update
- `handle_update_fee`
  - `internal_update_fee`
    - If this returns an error, `handle_error`

# Dust Handling

How does LDK think about dust values? These need to be considered for:
- Fee spike buffers
- Fees on offered HTLCs
- Dust exposure

`get_dust_exposure_limiting_feerate`:
- Returns a feerate that expresses the most aggressive fee rate we'd
  possibly want to consider.
- Used to sanity check fee rates we get from peers.
- This should be very conservative, but not so conservative that we
  allow a peer to give us an insane fee rate that burns everything we
  have to dust.

It's used in a few places in the codebase:

`get_available_balance_for_scope`:
- `htlc_stats` = `get_pending_htlc_stats`:
  - `get_dust_buffer_feerate` returns a feerate for dust assessments
    that considers the possibility that we might hit a fee spike
    - This function is only used for balance checks, so it can add
      some future-looking predictability to balances
  - If we use anchors, we don't have to consider this (because we don't
    have to add fees to our success/timeout txns)
  - Trims any HTLCs that are under the respective dust limits 
  - `excess_feerate_opt`:
    - Pulls any waiting fee rate update that's pending
    - Or falls back to our current channel fee rate
    - Subtracts the maximum feerate that we use to limit our peer
    - This provides a "marginal" feerate above our maximum
  - `extra_nondust_htlc_on_counterparty_tx_dust_exposure_msat`:
    - The dust exposure on their commitment + one more HTLC at this rate
  - If this marginal feerate is non-zero, we also add the marginal extra
    amount of fees to `on_counterparty_tx_dust_exosure_msat`
  - In summary, we're considering the possibility that our fees go up
    and we're above the feerate that we consider to be "safe" to not
    consider our funds to be siphoned off into fees.
- `htlc_stats.pending_outbound_htlcs_value_msat` is subtracted from
  our available `outbound_capacity_msat`
- `max_dust_exposure_msat` = `get_max_dust_exposure_msat(dust_exposure_limiting_feerate)`
  - There are two ways that we can limit dust exposure:
    - `FeeRateMultiplier`: don't allow dust greater than x * feerate
    - `FixedLimitMsat`: don't allow dust above configured limit
-  `htlc_success_dust_limit`, `htlc_timeout_dust_limit`
  - If anchors = `counterparty_dust_limit`, `holder_dust_limit`
  - If not anchors, needs to include success/timeout
- If the `extra_nondust_htlc_on_counterparty_tx_dust_exposure_msat` would
  push us over the dust limit in total fees, we can't add non-dust HTLCs
- Check both holder and commitment transactions to see whether one
  more HTLC would push us over dust exposure limits:
  - If they would, we clamp our available balance to only send below
    appropriate dust limit so that we don't go over limit
- If `remaining_msat_below_dust_limit`, we're below the dust limit but
  might have some sats left that we can send:
  - If we only have dust capacity left on our side we set our available
    capacity to be the remaining sub-dust limit (or itself if smaller)
  - If we have non-dust capacity left, we enforce that all HTLCs are
    at least the dust limit? Or our minimum HTLC size if above the
    dust limit

`update_add_htlc`:
- `htlc_stats` = `get_pending_htlc_stats` 
  - Doesn't actually use any of the dust checks, just amkes sure that 
    we don't go over any of our channel restrictions

`send_update_fee`:
- `htlc_stats` = `get_pending_htlc_stats`
- `get_max_dust_htlc_exposure_msat(dust_exposure_limiting_feerate)`
  - Applies a multiplier to the feerate to limit dust for new feerate

`update_fee`:
- Same as above

`can_accept_incoming_htlc`:
- `htlc_stats` = `get_pending_htlc_stats`:
- Checks dust exposure against `max_dust_exposure_msat` which uses the
  `get_dust_exposure_limiting_feerate` value to get the max msat of
  dust that we're okay with.

## Notes for V3

`dust_exposure_limiting_feerate`:
We never use the dust feerate for zero fee channels because dust is
independent of feerate in anchor channels (previously, a different fee
rate would impact whether a HTLC is dust or not because subtracting
fees off of it would potentially dip them below the dust limit).

It does seem to me that just setting this value to 1 sat/vbyte for
zero fee channels seems like a bad idea when we have dust protection
options that depend on the feerate? Although, those aren't ever really
used in zero comms because we don't hit the checks? We do in 
`can_accept_incoming_htlc`, for example.

It seems like we're dealing with one value in two different contexts:
- `get_max_dust_htlc_exposure_msat`: (if configured) uses the limit to
  calculate the total amount of dust that we'll allow ourselves. 
- `get_pending_htlc_stats`: to figure out whether one more HTLC will
  push us over the dust limit (but actually not really for V3, because
  the dust limit doesn't matter for us).

Q: Can we separate these two things?
-> Seems like at the very least we can pull out the max dust check
   into `htlc_stats` so that every single caller doesn't need to get
   and handle the dust thresholds.

Q: Is this fee rate currently dynamically set? If yes
  maybe we don't want to hard code it for zero fee?
- Yes. So when we calculate our fee exposure for other channels we'll
  get the maximum fee rate that we can think of for the current fee
  environment. Now we'll always hard-set that to 1 sat/vbyte.
  - But, comment notes that now that we're not depending on our fee rate
    for our dust amounts, we don't want to move anyway.
  - When it's for fee calculation we ignore it

Decision is:
[x] get_dust_exposure_limiting_feerate: update this to return an option
[x] get_max_dust_htlc_exposure_msat handles the option
- optionally refactor dust limit stuff

`get_available_balance_for_scope`:
----------------------------------
- `extra_htlc_dust_exposure` > `max_dust_exposure_msat`
- `on_counterparty_tx_dust_exposure_msat` + `htlc_success_dust_limit` > `max_dust_exposure_msat` +1
- `on_holder_tx_dust_exposure_msat` + `htlc_timeout_dust_limit` -1 > `max_dust_exposure_msat`

`send_update_fee`:
------------------
`on_counterparty_tx_dust_exposure_msat` > `max_dust_exposure_msat`
`on_holder_tx_dust_exposure_msat` > `max_dust_exposure_msat`

`update_fee`:
-------------
`on_counterparty_tx_dust_exposure_msat` > `max_dust_exposure_msat`
`on_holder_tx_dust_exposure_msat` > `max_dust_exposure_msat`

can_accept_incoming_htlc:
-------------------------
`on_counterparty_tx_dust_exposure_msat` > `max_dust_exposure_msat`
`htlc_success_dust_limit`

-> Moved the dust limits into a helper, can do further refactors if
   it's helpful.

We don't actually want to use the non-anchor feerate for zero fee
channels because that's a higher fee rate to make sure that the non
anchor channel gets in.

Check in on calculate_closing_fee_limits:
[Q]: How are we going to close out zero fee channels - we can't set a
fee on them if they have an anchor output? They're not allowed to have
fees if they have dust.

The `anchors_zero_fee_htlc_tx` is no longer aligned with the spec's
naming after https://github.com/lightning/bolts/pull/1092.

In `internal_open_channel` we'd also need to reject zero fee channels
by the same logic.

`build_commitment_stats` needs to have different anchor values.
`build_commitment_transaction`: do we still subtract anchor from funder?

`insert_non_htlc_outputs` needs to account for anchors
