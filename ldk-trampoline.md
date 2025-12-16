# Trampoline Routing

Top level notes for reviewing trampoline routing in LDK.
Top level tracker: https://github.com/lightningdevkit/rust-lightning/issues/2299

Mega PR: [3946](https://github.com/lightningdevkit/rust-lightning/pull/3976)

Broken up into parts:
[3983](https://github.com/lightningdevkit/rust-lightning/pull/3983)
[4029](https://github.com/lightningdevkit/rust-lightning/pull/4027)

## Sending to Trampoline

To send to a trampoline, the sending node must choose the trampoline(s)
that they support, and build an onion packet with these trampolines.

There are various payment helpers that call down to `send_payment_along_path`:
* `send_payment` -> `send_payment_along_path`
* `send_payment_with_route` -> `send_payment_along_path`
* `bolt11` / `bolt12` / `spontaneous` -> `send_payment_along_path`

Let's look at `send_payment_with_route`, because this allows us to
provide a `Route` (there aren't specialized trampoline APIs yet):
- Provide a `Route`:
  - Has a `Vec<Path>` (allows MPP)
```
path {
  hops: Vec<RouteHop>
    Hop {
      pubkey
      short_channel_id
      fee_msat (etc)
    } 

  blinded_tail: Option<BlindedTail>
    BlindedTail {
      trampoline_hops: Vec<TrampolineHop>
        TrampolineHop {
          pubkey: Pubkey,
          fee_msat,
          cltv_expiry_detla,
        }
      hops: Vec<BlindedHop>
        BlindedHop {
          blinded_node_id: Pubkey,
          encrypted_payload: Vec<u8>,
        }
      blinding_point: Pubkey,
      final_value_msat: u64,
    }
}
```
- Provide other standard payment info (hash, recipient fields etc)

Notes about our `Route`:
- When we don't have a `BlindedTail`, all of the `Path`'s final pubkey
  must be the same (ie, going to the same destination).
- When using `trampoline_hops`, the pubkey of the first trampoline must
  be the same as the last `RouteHop` in the `Path`

So, what would some paths look like?

Example 1: Trampoline without blinded paths (LDK doesn't support)
```
hops: [A, B, C]
blinded_tail / trampoline_hops: [C, E]
```

We're sending `A -> B -> C -{trampoline route}-> E`

Example 2: Trampoline with blinded path
```
hops: [A, B, C]
blinded_tail:
  hops: E (intro), B(F)
  trampoline_hops: C
```

We're sending `A -> B -> C -{trampoline route} -> E -> F`
Where the trampoline `C` is able to create a path to introduction point
`E`. The introduction `E` will have a `blinding_point` inside of its
payload that allows it to decrypt the destination `B(F)` to `F` and
forward the payment to `F`, sending the `blinding_point` in
`update_add_htlc` to `F` so that they can decrypt their data.

`send_payment_with_route`:
- Created a `FixedRouter` from our `Route` (just returns route)
- `self.pending_outbound_payments.send_payment` 
  - `self.send_payment_for_non_bolt_12_invoice`:
    - `route` = `find_initial_route`: returns selected route
    - `onion_session_priv` = `add_new_pending_payment`: (1)
    - `res` = `pay_route_internal` (2)

(1) `add_new_pending_payment`:
- Add `payment_id` to `pending_outbound_payments`
- `payment, onion_session_priv` = `create_pending_payment`
  - Fill `onion_session_priv` vec with random bytes
  - Insert the `session_privs` into the `PendingOutboundPayment` 
-> `onion_session_priv`

(2) `pay_route_internal`:
- Does some basic validation on the path 
- `results.push` `send_payment_along_path` (2.1)
- Iterate through results and return success/failure

(2.1) `send_payment_along_path` is a closure provided at the top
of our call stack, using `ChannelManager::send_payment_along_path`:
- `onion_packet, msat, cltv` =  `create_payment_onion` (2.1.1)
- Get first channel and validate it can send payment
  - `send_hltc_and_commit` with `HTLCSourceOutboundRoute`
  - `handle_new_monitor_update`

(2.1.1) `create_payment_onion` / `create_payment_onion_internal`:
Assuming that we have the following `Route`:
```
hops: [A, B, C]
blinded_tail:
  trampoline_hops: [C, F]
  blinded_tail: H, B(I), B(J)
```

This would go over the following route, where `{}` indicates a path
found by the tramoplines:
`A - B - C -{D - E}- F -{G}- H - B(I) - B(J)`

- `tramopoline_payloads` =`build_trampoline_onion_payloads` (2.1.1.1)
- `trampoline_session_priv` = `compute_trampoline_session_priv`
- `onion_keys` = `construct_trampoline_onion_packet`
- `trampoline_packet` = `construct_trampoline_onion_packet`
- `build_onion_payloads`
- `construct_onion_keys`
- `construct_onion_packet`

## 3983

### Enforce Trampoline Constraints

`check_trampoline_onion_constraints`:
- Accepts the `outer_hop_data` and the `cltv` and `amount` from
  the trampoline onion
- Checks that the amount is correct

We call this in:
- `create_fwd_pending_htlc_info`
  - `get_pending_htlc_info` (1)
  - `peel_payment_onion` (2)
- `create_recv_pending_htlc_info` 
  - `get_pending_htlc_info` (1)
  - `peel_payment_onion` (2)
  - `forwarding_channel_not_found` (3)

(1) `get_pending_htlc_info`:
- Called by `process_pending_update_add_htlcs`:
  - We decode our `UpdateAddHTLC`'s payload, returning
    - `Hop`: tells us what the next item is (receive/forward etc)
    - `NextPacketDetails`: outgoing values, scid or trampoline
  - Perform various checks on the htlc forward
  - Call `get_pending_htlc_info` to create forward or receive
    - Push this into `forward_htlcs` that'll be processed later

(2) `peel_payment_onion`:
- Only called in tests, possibly used to provide a onion handling util?

(3) `forwarding_channel_not_found`:
- If we have a `HTLCForwardInfo::AddHTLC`:
  - And it is a htlc that was supposed to be forwarded and a valid
    phantom ID:
    - Create phantom receive payload and then push it to our phantom
      receives

So, we are in `process_pending_update_add_htlcs` and we get some
trampoline payloads!
- Currently, we hit `can_forward_htlc` / `can_forward_htlc_to_outgoing_channel`:
  - We fail the HTLC out, not wanting to forward trampoline stuff   

Take a look at the tests!

What do we have before this:
- `test_trampoline_forward_payload_encoded_as_receive`
- `test_trampoline_single_hop_receive` (`do_test_trampoline_single_hop_receive`)
- `test_trampoline_unblinded_receive` (`do_test_trampoline_unblinded_receive`)
- `test_trampoline_forward_rejection`

After this we introduce 

(1) `test_trampoline_unblinded_receive_(underpay/normal)`:
- It modifies `do_test_trampoline_unblinded_receive` to only do underpay
  (vs previously was for success/fail)

(2) `test_trampoline_blinded_receive_enforced_constraint_cltv`
    `test_trampoline_enforced_constraint_cltv`
- It has its own helper, `do_test_trampoline_unblinded_receive_constraint_failure`

Q: Can we get away with a unified test helper for testing unblinded?
- Successful (all correct)
- Not successful (amount / cltv bad)

Underpayment:
- We modify the FinalOnionHopData to have the higher amount
- Depending on underpay, we settle or fail the htlc 

CLTV issues:
- If we want the CLTV to be too much, we manipulate the blinded_hop_cltv
- We assert on the failure accordingly 

Now, we do have different tests for blinded and un-blinded!
The goal is to have *unified* test helpers: one for unblinded, one for
blinded. They should each be able to settle or fail.

Comparison between:
(A) `do_test_trampoline_unblinded_receive_constraint_failure`
(B) `do_test_trampoline_unblinded_receive` 

Common values inline, differences marked by (A)/(B):
- Creates a three hop network and payment details
- (A) Sets up route:
  - Creates a real set of `TrampolineForwardTlvs`
  - Blinds the TLVs with real blinding point
  - Sets a manipulated cltv detla in `TrampolineHop`
- (B) Sets up route:
  - Sets an mock `hops` in `blinded_tail`  (because replacing anyway)
- Override randomness
- `send_payment_with_route`
- Creates replacement onion:
  - (A): with the `amt_msat` of the payment
  - (B): with the override amount for underpayment
- Constructs onion with new inner packet
- Check monitors
- `get_and_clear_pending_msg_events`
- Get update add and replace onion
- Generate `args` to pass along path from `PassAlongPathArgs::new`
  - (A): sets preimage, no claimable event, failure event
  - (B): sets preimage, no claimable event, failure event *if underpay*
         else just sets paymentsecret (this is the case it succeeds)
         -> We don't actually need this I think?
         [ ] Remove this!
- `do_pass_along_path` 
- Failure case:
  - `get_htlc_update_msgs(2/1)` / `1.handle_update_fail_htlc(2)`
  - `do_commitment_signed_dance(1, 2)` 
  - `get_htlc_update_msgs(1/0)` / `0.handle_update_fail_htlc(1)`
  - `do_commitment_signed_dance(0, 1)`
  - Match failure:
    - Get expected payment data
    - Set `LocalHTLCFailureReason`
    - `expect_payment_failed_conditions`
- Success case:
  - (A) Does not have a failure case
  - (B) `claim_payment`

-> Managed to combine these into one helper!

#### Blinded vs Unblinded

Q: Can I update `do_test_trampoline_unblinded_receive` to also cover the
blinded case?

What needs to change?
- Calculate shared trampoline secret
- Calculate blinding point + create proper receive tlvs
- Create a blinded path for carol with real values
- Don't replace the onion

Done - managed to combine tests into make a single test that covers
both blinded and unblinded.

## Forwarding Trampoline

What's already been done?
- Packet assembly from path
- Calculate onion length dynamically

To be done:
- Support forwarding as a trampoline payment to the next trampoline hop 
- Support MPP aggregating at intermediate hops
- Shorten CLTV (ensure it's relative)
- Support retrying a trampoline payment as an intermediate node

### 3976

This PR has all of Arik's original work with comments addressed (by
Maurice).

#### Create TrampolineForward HTLCSource Variant

We need to store more information for trampoline:
- The new session key that we're using for the route
- The IDs of previous HTLCs (now, we could have multiple htlcs arrive
  on our incoming link that we have to be responsible for settle/failing
  when we resolve on our outgoing link).
- `HTLCSource`: this enum is where we store additional info (eg if it's
  a forward vs our own payment).
  - This is tracked in `OutboundHTLCOutput`, which is where we keep all
    our info about our channel's outgoing htlcs (in
    `pending_outbound_htlcs`)
  - Written/read in channel (the oldschool way)

Use of this field:
- `fail_htlc_backwards_internal`:
  - Fail back each of the incoming htlcs when the outgoing fails
  - TODO: we get a failure back for our chosen onion route, but don't
    yet use it (because the sender doesn't care about our failed route).

We have two helpers here:
- `get_htlc_failure_from_blinded_failure_forward`:
  - Does blinded path voodoo if required
- `fail_htlc_backwards_from_forward`:
  - Puts failure in forward_htcs
  - Omits `HTLCHandlingFailed` event

- `claim_funds_internal`:
  - Success case where we claim our outgoing
  - Just do it for all of our trampoline htlcs

Another helper:
- `claim_funds_from_hop_internal`:
  - Does the "from hop claim" stuff that's common to regular and tramp

#### Expand HTLCDestination variants for Trampoline Forwards

- Just adds a variant to the API to notify trampoline failure

#### Find path and route to the next Trampoline node

- Add a `payment_id` to `TrampolineForward` to be able to track the
  status of our outgoing payment.
- Add an argument for `trampoline_forward_info` to `SendAlongPathArgs`
  so that we can include trampoline payloads.
- In `send_payment_along_path`:
  - Construct onion with trampoline payload if we have one
  - Create our `htlc_source` with appropriate trampoline info
  - `send_htlc_and_commit` to dispatch

`process_pending_update_add_htlcs`:
- Loops through all the `decode_update_add_htlcs`
- Decodes the outgoing packet
- `can_accept_incoming_htlc`: checks forwarding concerns
- For payloads that have an `outgoing_connector`:
  - `can_forward_htlc`: checks outgoing link's conditions for fwds
  - Trampoline: we don't know outgoing link, so don't check
- We push into `htlc_forwards`

`process_receive_htlcs`:
- This will be hit in `internal_process_pending_htlc_forwards`
  if we don't have an outgoing channel id (which is the case for tramp)
- Match to a trampoline value:
  - Create a `HTLCSource` that's a trampoline forward
  - Perform blinding ops + setup route parameters
  - `pending_outbound_payments.send_payment_for_trampoline_forward` 

`fail_htlc_backwards_internal`:
- `should_fail_back` = `pending_outbound_payments.trampoline_htlc_failed`
- If `should_fail_back`:
  - Fail each htlc back like we did before

`pending_outbound_payments.send_payment_for_trampoline_forward`:
- Finds a route for the onion payment
- `add_new_pending_payment` 
- `pay_route_internal`
- `handle_pay_route_err`

`trampoline_htlc_failed`:
- Decode the onion failure received
- Allow another shot if it's still retryable and we don't have
  the full amount in flight and it is a retryable payment

`create_trampoline_forward_onion`:
- Creates a `RecipientOnionFields::spontaneous_empty()`
- Builds onion payloads, adds onion payload
- Construct full onion packet

#### Thoughts

Q: What about all the fussy "we have two commitments" stuff?
- [ ] The following helper functions can be prefactored
  - `get_htlc_failure_from_blinded_failure_forward`
  - `fail_htlc_backwards_from_forward`
  - `claim_funds_from_hop_internal`
- Generally, the one massive claim function is hell and could use some
  cleanup
Q: Are we going to support multi-part outgoing?
Q: `new_trampoline_entry` has a blinding point added, but we never
  pass one in in `build_onion_payloads_callback`
Q: What's the blocking situation here?
Q: What happens if we have two trampoline payments with the same hash?
Q: Can the trampoline retry logic be de-duplicated?
Q: Is the `unreachable` in `create_trampoline_forward_onion` ok? 

I think this should be broken up into:
- Add trampoline `HTLCSource` and the prefactors that come with it
- Add routing of new payments

Maurice already had us most of the way for the refactors (ITO splitting
it up and breaking into functions).

Q: What do we need `hops` for in `HTLCSource::TrampolineForward`?

## PR Runthrough

Going through PR once over before opening up.

### add blinded forwarding failure helper function

get_htlc_failure_from_blinded_failure_forward:
- Takes a blinded failure which indicates whether we're in a blinded
  route (or not)
- The original `onion_error` which has our local failure
- Various shared secrets that we may need to use for error wrapping
- The HTLC ID that we're failing

### move forward fail back to fail_htlc_backwards_from_forward

fail_htlc_backwards_from_forward:
- Takes an `onion_error`, `failure_type` and `prev_channel_id` to
  report in the `HTLCHandlingFailed` event that's omitted
- The `short_channel_id` is the ID that we actually push to
  `forward_htlcs`, and the `failure` is the action that we're taking.

Pulling this out because we'll need to call it multiple times for
trampolines with multiple incoming HTLCs.

### move funds claim for forward into helper

claim_funds_from_hop_htlc_forward:
- Gets the monitor action we'll need when done
- Processes attribution data
- Calls `claim_funds_from_hop` with a big ole closure (`completion_action`)
  - `claim_funds_from_hop` -> `claim_mpp_part`

Questions here:
- Is it okay to have one event per incoming hltc for a single outgoing?
- How are we going to "split" our fees for one in / one out

### extract channelmonitor recover to external helper

- When we read the channelmanager from disk, we have to reconcile our
  view with the ChannelMonitor, because they aren't persisted in the
  same db transaction.

### add TrampolineForward variant to HTLCSource enum

- channelmonitor:
  - `get_htlc_balance`: return false for trampoline because it's not
    our payment.
  - `get_claimable_balance`: likewise, return false for trampoline
    because it is not our payment
- channelmanager:
  - We add the `HTLCSource` variant for trampoline storing the info
    we need to be able to resolve these.
  - We implement hash for this in a similar way to `OutboundRoute`
    (which does include the path)
  - Implement read and write for the variant (in legacy way):
    - read: 2 / hop data, secret, priv, hops 
    - write: 2 / hop data, secret, priv, hops
  - We add `todo!()` in a few places where we need to add handling which
    we will fill in in later commits:
    - [ ] ChannelManager / `SendHTLCId::from_source`
    - [x] ChannelManager / `fail_htlc_backwards_internal`
    - [x] ChannelManager / `claim_funds_internal`
    - [x] ChannelManager / read

Questions:
- Why do we need to store `hops` here? Check in the final boss PR.
- Why aren't we storing the full `Path` like we do for `OutboundRoute`?
  - We don't actually have it?

### add trampoline routing failure handling

- When we call `fail_htlc_backwards_internal`, add handling for
  trampoline htlcs, canceling

Question:
- Where is this called and are we going to call it in the right
  place for trampoline? (ie, could we accidentally do this when we're
  still waiting on an outbound to succeed??)
- How do we report this paymnt back to "mission control"?

### add trampoline routing payment claiming

- Claims funds for each incoming htlc

Q: What about splitting fees here? It's wrong! Need to take another look
   here.

### add channel monitor recovery for trampoline forwards

- Does the reconcile for each hop, we do this per-outgoing htlc so
  when we're sending payments so I think it's fine to do it one by
  one?

## Outstanding Implementation Questions

### How do we implement `SendHTLCId` for trampoline?
  - The remaining PR includes both a `session_priv` and
    `previous_hop_ids` in a `TrampolineForward` variant of `SendHTLCId`
  - We don't appear to use the hops anywhere in the downstream PR, and
    it's wasteful to include all the hops
  - For `OutboundRoute` we just use the `session_priv`
  - For `PreviousHopData` we include the `PreviousHopData`

Where do we use `SendHTLCId`?
- `update_holder_commitment_data(claimed_htlcs)`:
  - `counterparty_fulfilled_htlcs`:
    - insert id/preimage
    - used to re-populate outbound htlcs in channel manager on restart
-> So far as I can tell, this is just used as a unique identifier in
   a preimage map so we don't need all hops.

- `provide_secret`:
  - Prunes any preimages that we no longer need because they aren't
    in any of our commits.
  - In the MPP case, we want to have distinct IDs here so that we don't
    settle one outgoing HTLC and then forget about the preimage for
    the other. Only our downstream can settle back off chain anyway,
    so it doesn't matter if we know the preimage (except for onchain).

- `commitment_signed_update_monitor`
  - We add the `SendHTLCId` to `claimed_htlcs` which is pushed into
    either
    - `LatestHolderCommitment`:
      - `update_holder_commitment_data`
    - `LatestHolderCommitmentTxInfo`
      - `provide_latest_holder_commitment_tx`
        - `update_holder_commitment_data`

Q: What happens with MPP payments? Would we add multiple here?
- `get_all_current_outbound_htlcs` goes through all of our
  `counterparty_claimable_outpoints` which has `HTLCSource` stored in
  it. We then create a `htlc_id` from that `HTLCSource` and populate
  the preimage.
  - If we had multiple outgoing htlcs, they'll be uniquely identified
    by their `session_priv`
  - If we did this by prev scid/id, we'd be overwriting them?

Q: Where do we create a `HTLCSource::TrampolineForward`?
- `send_payment_along_path`:
  - `send_htlc_and_commit`:
    - `send_htlc`:
      - Pushes into `OutboundHTLCOutput` where it will be persisted
-> We set params to allow MPP payments, and `pay_route_internal` will
  dispatch them if we need it.

TL;DR:
- I think that it's okay to just use the session key for
  `TrampolineForward`'s `HTLCSentId` (the worst case is that we will
  duplicate some preimage information).
- For multiple outbound htlcs, we'll add one `TrampolineForward` per
  outbound htlc; they'll all have their own unique `session_priv` but
  will duplicate the previous outgoing hops that they list.

### Do we want a single event for claiming all trampoline inbound htlcs?

- A `PaymentForwarded` even is generated when a payment has been
  successfully forwarded through and a fee is earned.

For trampoline: when we have multiple outgoing HTLCs, we'll omit one `PaymentForwarded` event per incoming htlc that we settle back. Are we happy to:
1) Have multiple events for a single trampoline forward? This makes sense to me because the event references the previous channel.
2) Assign the whole fee earned to the first event, and have the following ones be zero fee. This is the simplest way of doing it (splitting between events is possible, but could have annoying rounding issues)
.

Q: I need to look at where we're calling `claim_funds_internal` to
figure out where we're calling this from for trampoline (I'm not so
sure what the `next_channel` stuff will be because we can have different
next channels.
- When we get an `update_fulfill_htlc` we'll call `claim_funds_internal`:
  - On the first claim, we'll claim all of the incoming HTLCs and omit
    a `PaymentForwarded` event
  - On the second claim:
    - `claim_funds_from_hop`
      - Same `hop_data` as previously
      - Possibly different 

Q: Why does `claim_funds_internal` have `forwarded_htlc_value_msat`
  as an option?
- We alwayas set `Some` in all call sites, and it has been like this
  since we added the param.

Asked Matt:
- We should change the event
- Only thing that matters is serialization compat (should be easy to
  go from one thing to a vec)
- Could also do a new event type, not fussy: main thing that matters
  is clearly communicating what happened. 

Q: if things are only `None` for versions < 0.1, is it safe for me to
   migrate them away from being `Option`? Specifically:
- 0.0.107 - June 2022
- 0.0.112 - April 2024
- 0.1 - Jan 15

A: We already have a caveat that we can't directly upgrade from some
   versions with pending HTLCs, so in that case it's okay but otherwise
   we still want to respect it.

Action:
- Need to change refactor of claim to take multiple HTLCs + migrate
  `PaymentForwarded`; this also fixes the splitting issue.
- This gets a bit tricky, because it's really geared towards a single
  channel.

`claim_funds_from_hop`:
- Creates a `HTLCClaimSource` from `prev_hop`
  - We need one of these per incoming htlc!
- `claim_mpp_part`:
  - Accepts a completion function which optionally returns a
    `MonitorUpdateCompletionAction`
  - Lock the incoming `per_peer_state` + lookup channel
  - `update_fulfill_htlc_and_commit` creates claim + commitment
  - If `NewClaim`:
    - Get the action from `completion_action` + push to actions
    - `handle_new_monitor_update` will process action
  

Q: Can we call `claim_funds_from_hop` and then omit a single event?
   Possibly _outside_ of the `claim_funds_from_hop` logic.
  - We only call this in two places `claim_payment_internal` and
    `claim_funds_from_hop_htlc_forward`.

Q: How do we set `definitely_duplicate` when we call our closure?
- `get_update_fulfill_htlc_and_commit`:
  - `get_update_fulfill_htlc`:
    - If the `pending_inbound_htlcs` is already `LocalRemoved`, it'll
      be counted as `DuplicateClaim`
      - Each trampoline claim will be new, so we'll omit an event for
        each

We're also going to call `claim_funds_internal` for _each_ incoming
HTLC, but we'll hit `DuplicateClaim` on the incoming htlcs and only
omit one event (because we've already claimed the outgoing ones).

SO! I think that having a single event that tells us all of the htlcs
that are involved with trampoline is _fine_. *But* right now we'll omit
one per incoming HTLC, which isn't very easy to understand (?).

Things we can deprecate option on (<0.0.123)
- prev_channel_id: 0.0.107
- next_channel_id: 0.0.107
- prev_user_channel_id: 0.0.122
- next_user_channel_id: none if the payment was settled via onchain

- prev_node_id: 0.1
- next_node_id: 0.1

Omitting only one event:
- `EmitEventAndFreeOtherChannel`:
  - `event` field has the `PaymentForwarded`
  - This is pushed in `handle_new_monitor_update_completion_actions` 
  - We then `handle_monitor_update_release` *if* we have a
    `downstream_counterparty_and_funding_outpoint` (we do set this
    in our event).
- This looks _pretty identical_ to `FeeOtherChannelImmediately`, except
  that we don't persist this option, so I'm worried about something
  getting lost?
-> Rather make the event optional and handle it accordingly

Fees question: we could have multiple htlcs spread across multiple
incoming channels.
- Just have to bite the bullet and grab lock on each channel quickly
  to read the fee amount (total in - total out). We only use the
  claimed amount for fowards _anyway_ so it's not too bad to change.
- We're always going to hit this issue because we're in a possible
  multi-channel setting (there's no refactor of claim that would fix
  this).

Q: What about only on duplicate preimage?
- This would murk things up for MPP a bit? Yes, we lose an event

Q: Does this change the need for a different event type?
- No, we still need to be able to provide a signal that we're not
  going to omit an event and that has to live in the
  `MonitorUpdateCompletionAction`

Steps:
- Make event optional
- Add lookup for HTLC amount
- Remove the claimed amt from completed_action

## Remaining Questions

- Need to check what events look like for the failure side? Do they
  map well?
- Why do we need to store hops (and why not path) for tramopline?
  -> Look at the remaining PR to see how we use it
- How do we report the outcome of the trampoline forward to our
  "mission control" once we have it.
- Check how we're calling `fail_htlc_internal` for trampoline, looking
  at the downstream PR.
- Should probably also look at the different commitments thing to make
  sure we always forward back everything we're expecting to
