# Test Vectors for V3 Channels

Looking at `outbound_commitment_test` which is a similar test that will
be a good example of how to set these things up.

- We use a `ldk_test_vectors` feature flag.
- We're able to create an `InMemorySigner` which has the fixed keys
- We make various assertions on the kes that we're using.

## Recap of Commitment Points

When we send `open_channel` we provide:
- `funding_pubkey`
- `recovation_basepoint`
- `payment_basepoint`
- `delayed_payment_basepoint`
- `htlc_basepoint`
- `first_per_commitment_point`

When the remote party accepts the channel, they provide the same. These
basepoints are used in bolt-03 to derive unique keys for each commitment.

We produce public keys from our points as follows:
`pubkey = basepoint + SHA256(per_commitment_point || basepoint) * G`

We can derive our privkeys when the basepoints are known as follows:
`privkey = basepoint_secret + SHA256(per_commitment_point || basepoint)`

This is trivial because:
- `pubkey = privkey * G`
- `basepoint = basepoint_secret * G`
- We just tweak our `basepoint_secret` with the commitment point

We then derive the following keys:
- `localpubkey`:
  - Derived from: `payment_basepoint`
  - Used in: ????
- `remote_pubkey`:
  - Equal to remote node's `payment_basepoint`
   - Used in: your commitment to pay the remote peer their balance
- `local_delayedpubkey`:
  - Derived from: `delayed_payment_basepoint`
  - Used in: our commitment to pay our balance out (after delay)
- `remote_delayed_pubkey`:
  - Derived from: remote node's `delayed_payment_basepoint`
  - Used in: their commitment to pay their balance out (after delay)
- `local_htlcpubkey`:
  - Derived from: `htlc_basepoint`
  - Used in:
    - Our offered htlcs for timeout transaction
    - Our received htlcs for success transaction
- `remote_htlcpubkey`:
  - Derived from: remote node's `htlc_basepoint`
  - Used in:
    - Our offered htlcs for success case
    - Our received htlcs for timeout case

We also have a blinded revocation key that is generated for each
party. Once we know the other party's `per_commitment_secret`, we (and
only we) are able to derive the public key:
- `revocationpubkey`
  - Derived from:
    - Our `revocation_basepoint`
    - Their `per_commitment_point` 
  - Used in:
    - `to_local` to pay revocation path
    - Our HTLCs for revocation path
    - Second stage HTLCs revocation path

When we revoke our previous commitment, we provide:
- `per_commitment_secret`: the secret for our previous `per_commitment_point`
- `next_per_commitment_point`: the next `commitment_point` that we'll use

Both parties exchange these so that we can have a fresh commitment secret
for every state update.

Q: What is `localpubkey` used for? The spec says that the remote party
   uses our `localpubkey` in their `to_remote` output that pays us back
   our funds. This doesn't make sense to me because we put their
  *untweaked* `remotepubkey` in our `to_remote` and this instruction
  says that they're going to put our tweaked `localpubkey`?

What do we have in the test vectors?
Local:
- [x] `funding_privkey`
- [x] `payment_basepoint_secret`
- [x] `delayed_payment_basepoint_secret`
- [x] `htlc_basepoint_secret`

Assuming that these are for the local node:
- `recovation_pubkey`
- `per_commitment_point`

Remote:
- [x] `funding_privkey`
- [x] `payment_basepoint_secret`
- [x] `htlc_basepoint_secret`

We do not have for the remote:
- `delayed_payment_basepoint_secret`

We are only creating the local node's commitment, with offered and
received htlcs. This means that we will will need to know:
- `revocationpubkey`
- `local_delayedpubkey`
- `remotepubkey` 
- `remote_htlcpubkey`
- `local_htlcpubkey`
- `local_delayedpubkey`

-> We have everything that we need, but it's possible that it doesn't
fit _so_ well with the way that LDK sets up transactions.

`InMemorySigner`:
- We want a `revocation_base_secret` to create our signer
- We use this `revocation_base_secret` to create the
  `RevocationBasepoint` that's produced by `pubkeys` to create
  `ChannelPublicKeys.revocation_basepoint`:
  - Used in open/accept channel messages
  - Also used to create an `InboundV1Channel`

Action:
- We need to overwrite this value if we use it!
- Override when we create `InboundV1Channel` (in the `OpenChannel` msg

Debugging difference between transactions:
- Our `to_local` is incorrect
- CLTV and `local_delayedpubkey` look fine, so we must be missing
  overwriting the revocation key somewhere?
