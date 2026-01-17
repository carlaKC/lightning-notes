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

## ln: handle trampoline without validating in process pending

It's okay to not process because they'll go from here to process
receives.

Claude Planning (read this!):


  Current Architecture

  1. The Trait System:

  The codebase uses a OnionPayload<'a, 'b> trait to abstract over two
  payload types:
  - OutboundOnionPayload - for regular (outer) onions
  - OutboundTrampolinePayload - for trampoline (inner) onions

  The trait provides factory methods:
  trait OnionPayload {
      fn new_forward(...)          // Intermediate hop
      fn new_receive(...)          // Final receive
      fn new_blinded_forward(...)  // Blinded intermediate
      fn new_blinded_receive(...)  // Blinded final
      fn new_trampoline_entry(...) // Trampoline entrypoint
  }

  2. The Payload Types (OutboundOnionPayload variants):
  - Forward - Regular intermediate hop
  - Receive - Final hop (BOLT11/BOLT12)
  - BlindedForward - Blinded intermediate
  - BlindedReceive - Blinded final
  - TrampolineEntrypoint - Final hop that contains trampoline packet
  - BlindedTrampolineEntrypoint - Trampoline entrypoint with blinding

  3. The Building Flow:

  create_payment_onion_internal()
    ↓
    IF blinded_tail.trampoline_hops exists:
      build_trampoline_onion_payloads() → builds inner trampoline onion
      ↓
      trampoline_packet_option = Some(packet)
    ↓
    build_onion_payloads(trampoline_packet_option)
      ↓
      build_onion_payloads_callback()
        ↓
        For each hop (reverse):
          if final hop (idx == 0):
            match blinded_tail:
              Some(DirectEntry) → BlindedReceive
              Some(TrampolineEntry) → TrampolineEntrypoint  ← ONLY if
  blinded_tail exists
              None → Receive
          else:
            Forward or BlindedForward

  The Problem

  When forwarding a trampoline payment:
  - You have a pre-built trampoline_packet (from incoming onion)
  - You DON'T have blinded_tail.trampoline_hops (you're not building
  the inner onion)
  - You need to create TrampolineEntrypoint as final hop
  - But build_onion_payloads_callback only creates TrampolineEntrypoint
   when blinded_tail exists (line 576-590)

  Current workaround: create_trampoline_forward_onion does
  post-processing to replace Receive with TrampolineEntrypoint

  Clean Refactoring Proposal

  Step 1: Create explicit final hop type enum

  /// Specifies what type of final hop to include in the onion
  pub(super) enum FinalHopSpec<'a> {
      /// Regular receive (BOLT11 with payment_secret)
      Receive,
      /// Keysend receive
      ReceiveKeysend,
      /// BOLT12 invoice_request receive
      ReceiveInvoiceRequest(&'a InvoiceRequest),
      /// Build inner trampoline onion from
  blinded_tail.trampoline_hops (originating)
      BuildTrampolineOnion,
      /// Use pre-built trampoline packet (forwarding)
      ForwardTrampolinePacket {
          packet: msgs::TrampolineOnionPacket,
          blinding_point: Option<PublicKey>,
      },
  }

  Step 2: Update build_onion_payloads signature

  pub(super) fn build_onion_payloads<'a>(
      path: &'a Path,
      total_msat: u64,
      recipient_onion: &'a RecipientOnionFields,  // payment_secret,
  metadata, custom_tlvs
      starting_htlc_offset: u32,
      keysend_preimage: Option<PaymentPreimage>,  // Keep for trait
  compatibility
      invoice_request: Option<&'a InvoiceRequest>, // Keep for trait
  compatibility
      final_hop: FinalHopSpec<'a>,               // NEW - explicit
  final hop type
  ) -> Result<(Vec<msgs::OutboundOnionPayload<'a>>, u64, u32),
  APIError>

  Step 3: Refactor BlindedTailDetails enum

  enum BlindedTailDetails<'a, I: Iterator<Item = &'a BlindedHop>> {
      /// Blinded path receive - use encrypted hops
      DirectEntry {
          hops: I,
          blinding_point: PublicKey,
          final_value_msat: u64,
          excess_final_cltv_expiry_delta: u32,
      },
      // Remove TrampolineEntry - no longer needed
  }

  Step 4: Update build_onion_payloads_callback logic

  fn build_onion_payloads_callback(..., final_hop: FinalHopSpec<'a>,
  ...) {
      for (idx, hop) in hops.rev().enumerate() {
          if idx == 0 {
              match (blinded_tail.take(), final_hop) {
                  // Blinded receive
                  (Some(DirectEntry { ... }), _) => {
                      // Create BlindedReceive payloads
                  },

                  // Build trampoline onion from
  blinded_tail.trampoline_hops
                  (None, FinalHopSpec::BuildTrampolineOnion) => {
                      let packet =
  build_inner_trampoline_onion(path.blinded_tail)?;
                      callback(OP::new_trampoline_entry(..., packet)?);
                  },

                  // Forward with pre-built trampoline packet
                  (None, FinalHopSpec::ForwardTrampolinePacket {
  packet, blinding_point }) => {
                      match blinding_point {
                          None => callback(OP::TrampolineEntrypoint {
  ... packet }),
                          Some(bp) =>
  callback(OP::BlindedTrampolineEntrypoint { ... packet,
  current_path_key: bp }),
                      }
                  },

                  // Regular receives
                  (None, FinalHopSpec::Receive) => {
                      callback(OP::new_receive(...)?);
                  },
                  (None, FinalHopSpec::ReceiveKeysend) => {
                      callback(OP::new_receive(...,
  keysend_preimage)?);
                  },
                  (None, FinalHopSpec::ReceiveInvoiceRequest(inv)) => {
                      // Use invoice_request in BlindedReceive if
  needed
                  },
              }
          }
      }
  }

  Step 5: Simplify create_payment_onion_internal

  pub(crate) fn create_payment_onion_internal<T: secp256k1::Signing>(
      ...
      trampoline_forward_info: Option<&NextTrampolineHopInfo>,
  ) -> Result<(msgs::OnionPacket, u64, u32), APIError> {

      let final_hop = if let Some(forward_info) =
  trampoline_forward_info {
          // Forwarding case - use pre-built packet
          FinalHopSpec::ForwardTrampolinePacket {
              packet: forward_info.onion_packet.clone(),
              blinding_point: forward_info.blinding_point,
          }
      } else if path.blinded_tail.as_ref().map_or(false, |t|
  !t.trampoline_hops.is_empty()) {
          // Originating case - build inner onion
          FinalHopSpec::BuildTrampolineOnion
      } else if keysend_preimage.is_some() {
          FinalHopSpec::ReceiveKeysend
      } else if invoice_request.is_some() {
          FinalHopSpec::ReceiveInvoiceRequest(invoice_request.unwrap())
      } else {
          FinalHopSpec::Receive
      };

      let (onion_payloads, htlc_msat, htlc_cltv) =
  build_onion_payloads(
          path, total_msat, recipient_onion, cur_block_height,
          keysend_preimage, invoice_request,
          final_hop,
      )?;

      // ... rest unchanged
  }

  Step 6: Eliminate create_trampoline_forward_onion

  No longer needed - everything goes through
  create_payment_onion_internal

  Step 7: Update channelmanager.rs

  // Remove branching - always use create_payment_onion
  let (onion_packet, htlc_msat, htlc_cltv) =
  onion_utils::create_payment_onion(
      &self.secp_ctx,
      &path,
      &session_priv,
      total_value,
      recipient_onion,
      cur_height,
      payment_hash,
      keysend_preimage,
      invoice_request,
      trampoline_forward_info,  // Pass through
      prng_seed,
  )?;

  Benefits

  1. Single code path - No branching between different onion creation
  functions
  2. Explicit intent - FinalHopSpec makes it crystal clear what type of
   final hop you want
  3. Type safety - Can't accidentally combine incompatible parameters
  4. No post-processing - Build the right payload directly
  5. Cleaner separation - Building vs forwarding trampoline onions is
  explicit
  6. Supports all cases - Blinded outer routing works for both
  originating and forwarding

  Does this approach make sense for your use case?
