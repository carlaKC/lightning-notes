# Transaction Batching

`BumpTransactionEvent`: contains the different types of transactions
that LDK might need to bump:
  - `ChannelClose`: local commitment of anchor channel.
  - `HTLCResolution`:  second stage htlc timeout/success.

`CoinSelectionSource`: trait that performs coin selection on our behalf.
- `select_confirmed_utxos`: 
  - `must_spend`: inputs that must be included in selection (RBF).
  - `must_pay_to`: outputs required (could be empty) 
- `sign_psbt`: provides full witness for inputs selected.

`WalletSource`: trait that provides a default implementation of
`CoinSelectionSource` when used along with `Wallet`.

`Wallet`: takes a `WalletSource` impl and tracks `locked_utxos`:
- `select_confirmed_utxos`:
  - Gets `list_confirmed_utxos` from `WalletSource`
  - Gets size of transaction (base + outputs + inputs)
  - For various different preferred configs, we perform our
    `select_confirmed_utxos_internal` (below):
    - We prefer not to have conflicting UTXO spends
    - We prefer not to tolerate very high fees
    - We'll give up on conflicts first, then give up on fees

- `select_confirmed_utxos_internal`:
  - Filter our UTXO set:
    - If it is in our locked set of UTXOs with a different claim ID
      tan our current claim *and* we're not trying to
      `force_conflicting_utxo_spend` (this is when we specifically
      want to conflict) then we skip it
    - Also apply a spending heuristic based on amount/feerate

`BumpTransactionEventHandler` is built to handle `BumpTransactionEvent`,
requiring `BroadcasterInterface`, `CoinSelectionSource` and 
`SignerProvider` to react to events.
- `handle_event`: just switches on type and calls different handlers.
- `handle_htlc_resolution`:
  - Given a set of htlc descriptors and locktime of tx
  - Get the channel type based on the first descriptor
  - Get the weights for success/timeout for that type
  - Add the HTLC output to our `must_spend` inputs, and set weight
    based on whether it's a timeout/claim
  - Add input/output to the tx we're building
  - Run coinselection for our chosen feerate
  - Add coins to our transaction
  - Get a psbt and sign it
  - Create our own htlc signatures
  - Broadcast transaction
- `handle_channel_close`: performs similar handling for closes
  - TODO: go through handling here.

Q: when do we omit these events?
Look for where we create `BumpTransactionEvent::HTLCResolution`:
- `channelmonitor::get_repeated_events`
  - `get_and_clear_pending_claim_events`
    - `ClaimEvent::BumpHTLC` 

The `channel_manager`'s `process_pending_events` will periodically
process these events with their handler, which results in our
consuming the `BumpTransactionEvent` and calling its handler (above).

These `ClaimEvent::BumpHTLC` are tracked in `OnchainTxHandler`'s
`pending_claim_events`:
- `update_claims_view_from_requests` pushes to this list
  - We get a series of `PackageTemplate` requests
  - We process them into allowable "batches" based on type/locktime/etc
  - `generate_claim` will produce the different types
  - Then we create `ClaimEvent::BumpHTLC`

A few other places do as well, but they're similar.
- `blocks_disconnected`
- `rebroadcast_pending_claims`

Q: Are the htlc descriptors always from the same channel?
- Yes, they have a single channel id in the event.

Q: Do we always provide at least one htlc descriptor?
- `BumpHTLC::htlcs` is a vec, so it could be empty but seems unlikely.

Q: How does re-selecting inputs after they're selected (but not used)
  impact things?
- We track `locked_utxos` in `Wallet`
- We filter out utxos that don't have the same claim ID (if not forcing)
  - Our claim ID will change every time, so we'll lock htlcs when we
    actually don't need them.
- Any utxos that are selected are added to the map under the claim ID

- We set `tolerate_high_network_feerates` to true _before_ we accept,
  so with this claim-ID changing we'll be more trigger-happy on high
  feerates

One downside with this approach that changes the claim id on each run is
that UTXOs will be locked that we don't actually end up using. When we
come back with a subset, these appear unavailable to our coin selection.
That makes it more likely that we'll bump up to `tolerate_high_network_feerates`.
Seems like we'd want a `free_claim` API to release the UTXOs that we
don't want, or a "readonly" version?
-> A static claim ID would help with this as well, then we don't need
   to worry about locking etc.

## Tasks to pick up

### Remove used UTXOs
There's a TODO which might be worth picking up on 410:
Do we care about cleaning this up once the UTXOs have a confirmed spend?
We can do so by checking whether any UTXOs that exist in the map are no
longer returned in `list_confirmed_utxos`.

### Use Bitcoin Utils

There's a TODO on 540:
Use fee estimation utils when we upgrade to bitcoin v0.30.0.
-> We're on 32 so this is ready to roll

`scale_by_witness_factor` looks helpful:
https://docs.rs/bitcoin/latest/bitcoin/struct.Weight.html#method.scale_by_witness_factor
-> We don't quite have what we need on hand here (in terms of types)
  eg, our inputs just provide their weight and we want to build a
  transaction which will handle weight itself

Something like:
```
let tx = Transaction {
    version: Version::TWO, // TODO: this isn't always correct.
    lock_time: todo!(),
    input: must_spend,
    output: must_pay_to,
}.weight();
```
