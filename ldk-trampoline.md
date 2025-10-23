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
- `self.pending_outbound_payments.send_payment` 
  - `self.send_payment_for_non_bolt_12_invoice`:
    - `route` = `find_initial_route`: (1)
    - `onion_session_priv` = `add_new_pending_payment`: (2)
    - `res` = `pay_route_internal` (3)

(1) `find_initial_route`:
