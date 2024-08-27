# Segwit

https://www.youtube.com/watch?v=JgNgnwF9hfY

Malleability: change the txid that a transaction has, breaking any
child transactions (because the outpoint refers to txid:index):
- Second party: your counterparty can choose a new r-value and change 
  their signature. If that (still valid) version of the signed 
  transaction is confirmed, your pre-signed child transactions are 
  broken.
- Third party: another party (not involved in signing) can change the
  txid by manipulating the signature or the script:
  - Signature: non-DER encoded signatures, these were not relayed early
    on in bitcoin's history (v0.8), but would still be accepted.
    Or negating the S value.
  - Input Script: change the script to be different but valid, eg 
    pushing superfluous data / not enforcing minimalif.

Why not BIP 62?
- It could not address second party malleability, which was the 
  interesting thing for lightning.

We moved witness data from our tx inputs scriptSig to the witness, 
and cleaned up the scripting issues so that the txid can't be malleated.

We have a marker flag so that segwit transactions are invalid to 
non-upgraded nodes.

[Quadratic Sighash](https://fjahr.com/posts/how-segwit-solved-the-quadratic-sighash-problem)
- We need to hash transaction data to get the digest that we need to 
  sign to produce signatures for bitcoin transactions.
- This hashing needs to be repeated by every validating node in the
  network when they validate signatures against transactions.
- When a transaction with n inputs is received:
  - Remove all other inputs from the transaction
  - Remove the candidate input script with the script of the output
    that it's trying to spend.
  - Double SHA256 the result

Every time you add an input, the amount of data that needs to be hashed
for each signature [increases](https://bitcoin.stackexchange.com/questions/87647/what-is-the-on2-signature-hashing-problem-and-how-does-segwit-solves-it/87653#87653).
-> For each of our n SIGHASH_ALL signatures, we have to hash n inputs.

[Detailed diagram.](https://en.bitcoin.it/w/images/en/7/70/Bitcoin_OpCheckSig_InDetail.png)

You can construct transactions that take pathologically long to 
validate; somebody [made one](https://rusty.ozlabs.org/2015/07/08/the-megatransaction-why-does-it-take-25-seconds.html)
on mainnet that took ~25s to validate.

Segwit updated the way that we calculate the digest to be signed, 
specifically makings sure that we don't have to hash _every_ input
for _every_ signature. We instead include the hash of previous outputs
and sequences for all signatures (committing to the full tx), but these
can be once-off precomputed to prevent quadratic issues.
-> If you implemented segwit very poorly it would still be quadratic.

Block Structure:
- Coinbase transaction now contains a OP_RETURN that has a merkle root
  of wtxids
- This output is contained in our merkle tree of transactions in the 
  block itself, which is covered by POW.
  -> This is NB: without somehow including witness data in our header,
     anyone can relay blocks with invalid witness data because we 
     haven't "expensively" committed to it.
- The witness stack for the coinbase transaction has a 32 byte nonce,
  its presence is enforced but there is no rectriction on its value.
  - We could use this for committing to a merkle tree of commitments,
    for example, a assumeutxo set.
  - This gives us a ton of extensibility for what a block can be used
    to commit to.

[Block headers](https://medium.com/fcats-blockchain-incubator/understanding-the-bitcoin-blockchain-header-a2b0db06b515)
