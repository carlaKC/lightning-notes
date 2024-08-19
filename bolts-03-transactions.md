# BOLT 03

- We order outputs by:
  - Amount (satoshis not msat)
  - Lexicographically: get the shortest `scriptPubKey` and compare that
    to the other value lexicographically (ie, by character)
  - By expiry height

This sorting is important for the order in which we send HTLC 
signatures, for example.

The current version of the specification uses P2WSH, future versions
will use P2TR (spec is WIP) - the spec omits the script on the witness
stack (it's always the last item).

The spec uses `<>` to represent an empty vector that satisfies the
[MINIMALIF standardness](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2016-August/013014.html) 
rule:
- Pre-Segwit, OP_IF and OP_NOTIF could be used to malleate transaction
  scripts:
  - Any non-zero value used as an input for IF/NOTIF could be replaced
    with another value (eg 0x01 = true could also be 0x02, resulting in
    the same value execution but a different script)
- Post-Segwit, the opcodes require an empty vector for false or `0x01` 
  for true, any other value will change so the script can't be

## Funding Transaction Output

Output script: 2 <pubkey1> <pubkey2> 2 OP_CHECKMULTISIG

## Commitment Transaction
Version: 2 (CSV soft fork)
Marker: 0x00 (input count field repurposed for segwit)
Flag: 0x01 (indicates segwit)
(see below for inputs and outputs)
Locktime: 00100000 xxxxxxxx xxxxxxxx xxxxxxxx

The 48 bit commitment number XORed with the lower 48 bits of:
SHA256(open_channel basepoint || accept_channel basepoint)
- This hides the number of commitments made on the channel
- Provides an index for nodes to quickly find their txns
- Only use the first 3 bytes of this value

Interpretation of locktime:
- 0 = any block
- < 500 mil = block height >= value of nlocktime
- >= 500 mil = interpreted as MTP

This value is > 500 mil, so interpreted as a timestamp. 2^29 is approx
Jan 1987 so we're "timelocked" to a value in the past. We're just 
stashing our commitment secret in nlocktime, effectively disabling it
with a very low value.

### Inputs
TxIn Count: 1
- Outpoint: [32] funding txid [4] index
- Input script: 0x00 (zero length for segwit)
- Sequence: [1 upper] 0x80 + [2 lower] obscured commitment nr
  10000000 xxxxxxxx xxxxxxxx xxxxxxxx (relative timelocks disabled)
- Witness: 0 <pk1_sig> <pk2_sig>

Bitcoin's interpretation of sequence for relative delays:
- Any value < 0xfffffffe indicates that the transaction siganls 
  replaceability (2 less than max)
- [31] Disable NSequence locks flag
- [22] Type Flag: 1 = time / 0 = blocks
- [15-0] Value Flag = 512 * v / v blocks

The number of items in the witness structure is implied by the number
of inputs. Each stack starts with the count of the elements on the
stack, and a size prefix per item. Legacy inputs just have a 0x00
witness stack.

### Outputs
All local outputs must have a to_self_delay which allows the remote 
party to claim revocation. This is done in a "second stage" transaction
for HTLCs because we first need to resolve the absolute timeout/ 
preimage claim (so that payments can resolve within an absolute timline),
then we handle relative revocation. All values are rounded down to the
closest satoshi value, and are assigned to fees if they're less than
the commitment holder's dust limit.

TxOutCount: depends on state of transaction
[8]: amount
[L]: output script length
[L]: output script

#### to_local:
```
OP_IF
    # Penalty transaction
    <revocationpubkey>
OP_ELSE
    `to_self_delay`
    OP_CHECKSEQUENCEVERIFY
    OP_DROP
    <local_delayedpubkey>
OP_ENDIF
OP_CHECKSIG
```

Spending revocation: <revocation_sig> true
Spending after csv: <local_delayedsig> <> (and nSequence = to_self_delay)
- CSV compares the value in the script to the nSequence of the input
- nSequence on the input is a transaction-level relative delay, so the
  tx is only valid after nSequence blocks/time has passed, which means
  that the input is only spendable after nSequence
  - If you created a spend without nSequence set, the transaction 
    would immediately be spendable but CSV would fail

#### to_remote:
Anchor channels are encumbered by a CSV 1, to prevent unconfirmed spends
from any output other than the anchors. This allows us to use the CPFP
carve out, which would otherwise be "taken up" by spends from other 
outputs (as it's specifically designed to allow "just one more" when
we have two spendable outputs.

```
<remotepubkey> OP_CHECKSIGVERIFY 1 OP_CHECKSEQUENCEVERIFY
```

Spendable with: <remote_sig> and `nSequence=1`

#### to_local_anchor / to_remote_anchor
```
<local_funding_pubkey/remote_funding_pubkey> OP_CHECKSIG OP_IFDUP
OP_NOTIF
    OP_16 OP_CHECKSEQUENCEVERIFY
OP_ENDIF
```

We always add an anchor, unless there are no htlcs and one of the 
parties balance is below the dust limit (only add an anchor for the
party that actually has balance, as there's nothing for the other party
to claim).

Claim your anchor witness: <local/remote sig>
Claim after 16 blocks: <> (and nSequence = 16)

#### Offered HTLCs
```
# To remote node with revocation key
OP_DUP OP_HASH160 <RIPEMD160(SHA256(revocationpubkey))> OP_EQUAL
OP_IF
    OP_CHECKSIG
OP_ELSE
    <remote_htlcpubkey> OP_SWAP OP_SIZE 32 OP_EQUAL
    OP_NOTIF
        # To local node via HTLC-timeout transaction (timelocked).
        OP_DROP 2 OP_SWAP <local_htlcpubkey> 2 OP_CHECKMULTISIG
    OP_ELSE
        # To remote node with preimage.
        OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
        OP_CHECKSIG
    OP_ENDIF
    1 OP_CHECKSEQUENCEVERIFY OP_DROP
OP_ENDIF
```

Remote spend with preimage: <remotehtlcsig><preimage> (nSequence 1)
- Not encumbered, because this is not their commit

Remote revocation: <revocation_sig><revocationpubkey>
- Does not need nSequence because we don't CSV this path

##### Timeout transaction:
Version: 2 (CSV soft fork)
Locktime: cltv_expiry (CLTV check)

TxIn Count: 1
- Outpoint: commitment transaction: htlc output index
- Sequence: 1 (anchors)
- Script bytes: 0 (segwit)
- Witness: 0 <remotehtlcsig> <localhtlcsig> 

TxOut Count: 1
- Amount: HTLC amount rounded to satoshis - fees
- Witness program: descried below

#### Received HTLCs
```
# To remote node with revocation key
OP_DUP OP_HASH160 <RIPEMD160(SHA256(revocationpubkey))> OP_EQUAL
OP_IF
    OP_CHECKSIG
OP_ELSE
    <remote_htlcpubkey> OP_SWAP OP_SIZE 32 OP_EQUAL
    OP_IF
        # To local node via HTLC-success transaction.
        OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
        2 OP_SWAP <local_htlcpubkey> 2 OP_CHECKMULTISIG
    OP_ELSE
        # To remote node after timeout.
        OP_DROP <cltv_expiry> OP_CHECKLOCKTIMEVERIFY OP_DROP
        OP_CHECKSIG
    OP_ENDIF
    1 OP_CHECKSEQUENCEVERIFY OP_DROP
OP_ENDIF
```

Remote timeout with: <remotehtlcsig> <> (nSequence 1)
- Not encumbered, because this is not their commit

Remote revocation: <revocation_sig><revocationpubkey>
- Does not need nSequence because we don't have CSV in this path

##### Success Tx
Version: 2 (CSV soft fork)
Locktime: 0 for success, cltv_expiry for timeout

TxIn Count: 1
- Outpoint: commitment transaction: htlc output index
- Sequence: 1 (anchors)
- Script bytes: 0 (segwit)
- Witness: 0 <remotehtlcsig> <localhtlcsig> <preimage>

TxOut Count: 1
- Amount: HTLC amount rounded to satoshis - fees
- Witness program: as below

#### Timeout and Success Txns
The witness program for timeout and success transactions is the same, 
we're already decided the outcome of the HTLC so now all we have to do
is impose a delay on the local node claiming those funds so that the 
remote has a chance for justice if required.

```
OP_IF
    # Penalty transaction
    <revocationpubkey>
OP_ELSE
    `to_self_delay`
    OP_CHECKSEQUENCEVERIFY
    OP_DROP
    <local_delayedpubkey>
OP_ENDIF
OP_CHECKSIG
```

Remote revocation: <revocation_sig> 1
Local node: <local_delayedsig> 0 (nSequence = to_self_delay)

SIGHASH_SINGLE | SIGHASH_ANYONECANPAY is used for these transactions.

## Closing Transaction
Version: 2
Locktime: 0

TxInCount: 1
- Outpoint: commitment tx:funding index
- Sequence: 0xFFFFFFFF (no RBF) 
- Script: 0 (segwit)
- Witness: 0 <pk_1_sig> <pk_2_sig>

TxOut Count: 1 or 2 (depends on final balances)
- Amount: balance to party
- Script: as specified in scriptpubkey of shutdown message

Read until: https://github.com/lightning/bolts/blob/master/03-transactions.md#fees

## Questions

1. Do we actually use the sequence index for scanning in impl?

2. Do we have to set nSequence for commitment transaction inputs to the
hidden commitment number? It's already in nlocktime.

## TRUC Updates
* Remove CSV 1 from`to_local` and `to_remote` and htlcs
* No longer need to set nSequence to 1 on spending txns
* Anchor update:
* Should we stop mucking with sequence values? Bitcoin can't use bits 
  16-21 for an upgrade because we fill it with our commit sequence.
  We already set this value in our nlocktime so why do we need it in 
  all our inputs?
