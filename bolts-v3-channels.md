# V3 Channel Upgrade

V3 gives us a lot of protection against pinning attacks. To upgrade, we
need:
- A new commitment format that uses V3
- Reserves/bumping available that can handle zero fee channels
- [to deploy] V3 standard on the network (should happen soon)
- [optionally] A method to upgrade channels on the fly

## Transaction Format

[Lightning Transactions with V3](https://delvingbitcoin.org/t/lightning-transactions-with-v3-and-ephemeral-anchors/418)
- On chain fees are tightly coupled with channel capacity, because
  they are currently kept up to date with the chain and subtracted
  from the funder's balance.
  - When fees go up, the capacity of lightning goes down!
- HTLCs that are dust also go into fees temporarily
- Proposed changes for V3:
  - Update transaction type to 3
  - Replace anchors with a single ephemeral anchor, equal to zero or
    the sum of all dust HTLCs
  - Remove CSV from all outputs (don't need carve out anymore)
  - Remove `update_fee` since it's no longer needed

This decouples LN from on-chain fees. This is particularly nasty when
we run into things like [fee spike buffer](https://github.com/lightning/bolts/pull/740):

[Discussion in Jan/25](https://github.com/lightning/bolts/issues/1221#issuecomment-2621162542):

Timeout and success transactions:
- Signature from peer: `SIGHASH_SINGLE | SIGHASH_ANYONE_CAN_PAY`
  - Allows us to combine this transaction with others
  - This is required because the transactions are zero fee
- Local signature: `SIGHASH_ALL` so that nobody else can attach junk
  to the transaction

Prior to this conversation, it was thought that we only need to upgrade
the commitment transaction and timeout transaction to V3, because a node
with the preimage claim would be incentivised to just claim the HTLC.
However, if we leave the success txn as V2, nodes will be vulnerable to
the following jamming attack via the success txns:

For pinning, consider the following forwards:
A ----(expiry =T)----> M 
A ---(expiry = T+5) -> M

* Mallory knows the preimages for both HTLCs, but withholds them
* At T-1 Alice must broadcast the commitment to claim HTLC expiring at T
* Alice includes a child transaction bumping the commitment transaction
  * The commitment is V3, so Mallory can't pin it
  * It gets confirmed in block T
* At T+4, Alice broadcasts her timeout transaction for HTLC T+5
* Mallory broadcasts a preimage claim for the same HTLC, attaching a 
  bunch of junk to pin it
  - She got a success txn sig from Alice with `SIGHASH_ANYONE_CAN_PAY`,
    so she can attach whatever she wants to it (which an honest peer
    would do to bump it, she does it to pin it).
  * It's not V3, so it's going to be expensive for Alice to replace
* If Mallory can keep Alice's timeout out of mempools, such that her 
  preimage claim succeeds, she can steal funds.

This works as follows:

M --(expiry T+10) --> A --(expiry T+5)--> M

* Mallory routes payments through Alice where she's in control of both
  sides (these can be large HTLCs, depending on what Alice allows in 
  flight, defaults are pretty big rn).
* Mallory performs the above pinning attack, making sure that mempools
  contain her undesirably low fee preimage claim of the HTLC at block
  T+5 rather than Alice's timeout claim
* Mallory managed to get her success claim to linger for 5 blocks
  until T+10 is reached, and times out her HTLC on M -> A
* Eventually her preimage claim confirms and she's pulled double the
  funds from Alice
  * She could possibly also RBF her pin (which would be expensive, but
    probably worth it for the large amount she can extract)

Making HTLCs V3 is challenging because we need to update our commitment
dance - you don't know beforehand what you should be signing?

## Pinning Discussion

