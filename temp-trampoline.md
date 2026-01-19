Didn't push temp file form work PC, creating this in the meantime.

`send_payment_for_trampoline_forward`
- Create a new payment secret for payment to next trampoline, and
  a recipient onion that just has a payment secret.
- We then query for a route to the destination.
- `add_new_pending_payment`:
  - Add an entry to `pending_outbound_payments`
  - Creates a `Retryable` payment
  - Generate the onion private keys for the chosen path
- `pay_route_internal`:
  - Performs some checks on the route
  - Calls `send_payment_along_path` which actually dispatches
  - `handle_pay_route_err` will perform any state updates needed
    if we can't dispatch the payment
  - We return an appropriate error for the payment

- [x] Added validation to `send_payment_internal` to make sure we only
  have forwarding fields in the payload.

- [x] Moving up `send_payment_along_path` change so that it's usable
  before our trampoline send commit.

- Creates a session private key for the payment being dispatched
- Create an `onion_packet, htlc_msat, htlc_cltv` for the info (1)
- Gets the channel we're sending over
- Creates the HTLC source (using `previous_hop_data` that we put in
  `trampoline_forward_info`.
- `send_htlc_and_commit` to dispatch the HTLC
- Does all the regular error handling 

(1) `create_trampoline_forward_onion`
- Creates an outer onion which pays the final amount in the path
  to the trampoline node that's receive it
- [ ] Takes a `keysend_preimage` that we shouldn't have?
- Builds the onion path for the outer onion using `final_value_msat`
- Creates recipient data for the trampoline
- Looks at the last onion paylaod we've created (must be a receive)
  - If unblinded, we add `TrampolineEntrypoint`
  - If blinded, we add `BlindedTrampolineEntrypoint`
- We overwrite the `last_payload`
- Finally, create onion keys and packet with the path and onion
  payloads provided

Q: what's the difference between `path.final_value_msat` and the 
  `total_msat` that's passed in here?
Q: would we ever pass in a `keysend_preimage` here?

(1) `create_payment_onion` (calls straight to `create_payment_onion_internal`:
- If there is a blinded tail
  - If there are trampoline payments
  - `build_trampoline_onion_payloads`
  - Create session private keys for trampoline
  - Construct onion keys for trampoline
  - Create onion packets for trampoline w/ keys
- `build_onion_payloads`
- Finally, construct onion keys and onion packet and payloads

- [ ] Issue:
- When we're the ones to build the trampoline payloads, we only
  support them if they're blinded
- When we receive the already built payload in a forwarding scenario,
  we have to cover both cases.

Going back to look into the payload types we support, I don't love the
claude suggestion for refactor here.

The types of outer onions we receive `InboundOnionPayload`:
- `Forward`: regular, no blinding
- `TrampolineEntrypoint`:
  - Could have `FinalOnionHopData` if if we've been MPP-ed to
  - Contains an inner onion packet (which we unwrawp)
  - `current_path_key`: we need to include this for the next node to
    unwrap their trampoline onion
  -> Basically, end of the outer onion time to do trampoline
- `Receive`: regular receive
- `BlindedForward`: blinded forwarding node gets routing info
- `BlindedReceive`: blinded recipient gets receive info

Types of inner trampoline onions we receive:
- `Forward`: we've been given the next node id to forward to
- `BlindedForward`: we're forwarding and get our routing info out of
  an encrypted data blob.
- `Receive`: we're the final recipient, but we looked like a trampoline
  to the previous node.
- `BlindedReceive`: we're the final recipient, and we got our recipient
  information out of encrypted data.

The types of ourter onions we create `OutboundOnionPayload`:
- `Forward`: regular forwading hop
- `TrampolineEntrypoint`: this is a trampoline and it gets the inner
  onions in its payload.
- `BlindedTrampolineEntrypoint`: this is a trampoline and it gets
  inner onions in its payload, along with a blinding point to help
  decrypt the inner onions.
- `Receive`:  regular receive
- `BlindedForward`: include encrypted tlvs for this node which has the
  forwarding information that it needs.
- `BlindedReceive`: was part of a blinded route, include final info 

The types of inner onions we create: `OutbountTrampolinePayload`:
- `Forward` + `Receive` regular fwd/recv inter-trampoline
  - Note: we don't support receiving unblinded
- `LegacyBlindedPathEntry`: end of trampoline, can send to blinded paths
- `BlindedForward`: include encrypted tlvs for node to get fwd info
- `BlindedReceive`: was part of a blinded route, include final info

Going back to our problem:
- `create_payment_onion_internal`:
  - Will create trampoline onions _if_ we have them in a blinded tail
  - Builds outer onion which will have these trampoline payloads in the
    last hop.

- `create_trampoline_forward_onion`:
  - Builds outer onion with the trampoline onion packet, creating a
    fake recipient onion for the final node
  - Gets the last onion payload that we've just built, which is a
    regular receive onion
    - Replaces it with either a `TrampolineEntrypoint` or a
      `BlindedTrampolineEntrypoint`

Q: Why can't we have trampoline packets in our path for inter-trampoline?
- We're picking a destination, and we don't have a blinded path from
  them to pay to.
- If we're a trapoline paying to a blinded path we have a
  `LegacyBlindedPathEntry` which isn't dealt with as a forwad (?)

Q: What are all the different case we need to handle?

Sending:
- Last trampoline is recipient:
  - Unblinded: sender includes final hop details in trampoline onion
  - Blinded: sender includes encrypted data in trampoline onion
- Exit to blinded path:
  - Sender receives a blinded path
  - Instructs last trampoline to route to introduction node
  - Puts blinded path information inside of subsequent trampoline
    payloads (blinded forward + blinded receive).

`build_onion_payloads_callback`:
- If we have `BlindedTailDetails::DirectEntry`:
  - Create a blinded receive at the end of the route
  - Create blinded forwards up until it, providing blinding point to
    the first one (the introduction node)
- If we have `BlindedTailDetails::TrampolineEntry`:
  - If there's a blinding point present, create `BlindedTrampolineEntrypoint`
  - Otherwise create `TrampolineEntrypoint`

Key difference:
- `DirectEntry`: we're just using a blinded path, and we should push
  multiple hops into our path for blinded forwards.
- `TrampolineEntry`: we're using trampoline forwading, and should push
  a trampoline onion which has already wrapped each of our blinded
  payloads in subsequent inner onions.

Note: LDK only supports the case where we're given a blinded path and
put the information inside of the trampoline onions, using each hop as
a blinded trampoline.

Thoughts:
- We should just be able to provide the right type of iter and we'll be
  fine? When we have a pre-built trampoline packet it's just a
  TrampolineEntrypoint?
- we need to call `build_onion_payloads_callback` with a TrampolineEntrypoint

Forwarding:
- When we receive a payload with a trampoline onion inside of it, we
  decode the inner onion and distinguish between the following:
  - `TrampolineForward`: given pubkey of next node
  - `BlindedFoward`: given forwarding details inside of blinded data
  - `Receive`: we're receiving and the person who sent knows it
  - `BlindedRecieve`: we've received via a blinded path
  - We fail if:
    - We get a forward with no next hop data
    - We get a receive with next hop data
- Regardless of whether we got a `TrampolineForward` or a
  `TrampolineBlindedForward`, we'll create `NextPacketDetails` with
  the pubkey of the next node in it.

Note: LDK doesn't currently support the case where you just throw some
blinded paths into the *outer* onion and then make a payment to them.
This is a workaround for when the recipient doesn't support trampoline.
I'm not sure whether we need to do this?

Note: For now, I'm going to ignore the above case and just work with
the "everyone supports trampoline" case.

Issue:
- We only add a `TrampolineEntry` if we've got a blinded path and there
  is Some `trampoline_packet`.
- For forwading, we may have a `trampoline_packet` which doesn't have
  any blinded path attached to it.

Q: I think that pairing the blinded path with trampoline was a mistake
that comes from only wanting to support them together in receives?
- In a world where 

## ln: handle trampoline without validating in process pending

It's okay to not process because they'll go from here to process
receives.

