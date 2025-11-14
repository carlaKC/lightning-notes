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
