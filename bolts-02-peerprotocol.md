# Bolt 02

## Channel State Update

Nodes exchange their `first_per_commitment_point` in `open_channel` and
`accept_channel` respectively, which is used to create the first
commitment transaction (spending from funding tx before it's signed to
return original balances).

When the channel has confirmed, `channel_ready` transmits the 
`second_per_commitment_point`, so that the nodes have the material
required to create their next commitment. Both nodes send this message
once they're happy that the funding transaction has reached sufficient
depth.

Commitment numbers are only explicitly transmitted during re-establish,
they are otherwise explicit. They start at 0, and the commitment signed
during channel establish bumps this number to 1.

Once we have updates on the channel, we provide the remote node with a
signature for their new commitment with `commitment_signed`. The peer
responds with `revoke_and_ack`, which provides both their 
`per_commitment_secret`.

## Channel Re-Establish

Used to get two nodes back on the same page about their views of the
current channel state. This is required because you don't know what 
the last message your peer saw was - for example, if you send an 
UpdateAddHtlc and then your connection is lost, you don't know whether
they received it or not before they went down.

To re-establish channels we transmit:
- next_commitment_number
- next_revocation_number
- your_last_per_commitment_secret
- my_current_per_commitment_point

Disconnection:
- Reverse any updates sent by the other side that we have not received
  a `commitment_signed` message for

Reconnection:
- MUST send `channel_reestablish` before sending any other messages for
  the channel (we can send _node_ level messages, ie init)
- MUST wait to receive `channel_reestablish` before sending any other
  messages for the channel

This ensures that we're both resuming on the same page, and don't have
to deal with new messages until we're resumed.

What are the various states we can be in?
=========================================

Before the channel's funding transaction has been signed, nodes will 
just forget about the channel if they disconnect.

### Accepting node has sent `funding_signed`:
- MUST remember the channel for reconnection
- It's possible that the remote node didn't get this message, and did
  not broadcast the funding transaction.
  - In this case, we'll eventually forget the channel because we never
    receive `channel_ready` from the remote peer.
- If the remote node did receive the `funding_signed` and then 
  disconnect, they'll just forget about the channel immediately (if 
  they didn't broadcast the funding tx).

Re-establish:
- The accepting node will send channel_reestablish, but the opening node
  will not (it forgot the channel).
- The accepting node will eventually timeout the channel.

### Opening node has broadcast `funding_tx`:
- It's impossible to make remembering broadcast and broadcast atomic:
  - You either have to write to db then broadcast, or broadcast then
    write to db. You can always go down between these non-atomic ops,
    and there's no way to rollback a tx broadcast (like you can w/ a db).
- Once the funding tx has been broadcast, we remember the channel and
  we know that our peer will have remembered it because the only way we
  get here is that they gave us `funding_signed`.

Re-establish: 
- The accepting and opening node will both send `channel_reestablish`
  with next_commitment_number=1, because they have both got their first
  commitment signed, and next_revocation_number=0 because nothing has
  been revoked yet (`your_last_per_commitment_secret` is zero, because
  you don't have one to give yet).
- The `next_commitment_number` is 1 for both nodes, so it's possible
  that we haven't yet sent `channel_ready`; we must retransmit our 
  existing message so that we can progress the channel (it's ignored if
  we did in fact send it before disconnection).
- Setting `your_last_per_commitment_secret` provides your remote peer
  with the ability to _prove_ that you've lost state; if they can give
  you a valid "future secret" by your current reckoning, then you have
  lost state.

### Commitment Signed Sent but not Received
From this point on, we'll think about all channel updates in the 
perspective of A and B, because it doesn't matter who opened the channel
anymore. *Everything expressed from B's perspective*.

Flow of messages:
```
A --> commitment_signed (x) B
```
- B has received updates, but no commitment signed so will reverse all
  of the updates that it has received from A.

Re-establish:
A last signed commitment = 2
B last signed commitment = 1

A / next_commitment_number: 2 (commitment after the one from funding)
A / next_revocation_number: 1 (expecting first commitment's revocation)

B / next_commitment_number: 2 (commitment after the one from funding)
B / next_revocation_number: 1 (expecting first commitment's revocation)

- The `next_commitment_number` from B is the one that A just sent, so
  B did not receive and process the `commitment_signed`. 
- A will re-use the number for their next `commitment_signed`, because
  it knows that B did not receive it.

### Commitment Signed Send and Recieved

Flow of messages:
```
A --> commitment_signed --> B
```
- B has received updates and a commitment signed, so will not forget
  the updates on disconnect.

Re-establish:
A last signed commitment = 2
B last signed commitment = 1

A / next_commitment_number: 2
A / next_revocation_number: 1

B / next_commitment_number: 3
B / next_revocation_number: 1

- The `next_commitment_number` from B is 1+last value, so A knows that
  B received the `commitment_signed` that they sent.
  - When the value is your current value +1, this means they receive it!
- A will increment their commitment number for `commitment_signed`
  because they know that B has processed their commitment_signed.

### Commitment Signed Disagreement
Flow of messages:
```
A --> commitment_signed --> B
```

There are two possible cases here:
- A has forgotten that they sent commitment signed
- B has lied about receiving a commitment signed

Re-establish:
A last signed commitment = 1 (should be 2)
B last signed commitment = 1

A / next_commitment_number: 2
A / next_revocation_number: 0

B / next_commitment_number: 3
B / next_revocation_number: 0

A can't be sure of what's the truth here, so they fail the channel but
take no action.

### Commitment Signed Summary
switch
    case next_commitment_number = last_commitment_number
        retransmit

    case next_commitment_number = last_commitment_number +1
        no action, up to date

    case next_commitment_number < last_commitment_number:
        the sending node has lost data (or lied)

    case next_commitment_number > last_commitment_number +1:
        we lost our last commit_signed OR remote is lying about it

### Revoke and Ack Sent but not Received
Flow of messages:
```
A --> commitment_signed --> B
A  (x)  revoke_and_ack  <-- B
```

Re-establish:
A last revoked = 0
B last revoked = 1

A / next_commitment_number: 2
`A / next_revocation_number: 1

B / next_commitment_number: 3
B / next_revocation_number: 1 

- B has received A's `commitment_signed`, so they know this has been
  mutually accounted for.
- The `next_revocation_number` that B received from A is equal to the
  revocation they just sent, they they will re-transmit it.

### Revoke and Ack Send and Received
Flow of messages:
```
A --> commitment_signed --> B
A <--  revoke_and_ack  <-- B
```

Re-establish:
A last signed commit = 2
B last signed commit = 1

A last revoked = 0
B last revoked = 1

A / next_commitment_number: 2
A / next_revocation_number: 2

B / next_commitment_number: 3
B / next_revocation_number: 2 

- A's `next_revocation_number` is not equal to the last `revoke_and_ack`
  that B sent, so they know that the message has been received.

### Revoke and Ack Summary

switch:
    case next_revocation_number == last_revoked_commit:
        retransmit revoke_and_ack

    case next_revocation_number == last_revoked_commit +1:
        no action, up to date

    case next_revocation_number < last_revoked_commit:
        the sending node has lost data
        
    case next_revocation_number > last_revoked_commit +1:
        if they have a valid recvocation secret for commit numer 
        next_revocation_number -1, we know that we have lost state

Commitment Number - Happy case
==============================
`next_commitment_number` = your last `commitment_signed`
- They have not received your `commitment_signed`, retransmit

`next_commitment_number` = your last `commitment_signed` + 1
- You are on the same page re the commitment you've most recently signed

Commitment Number - Bad case
============================
`next_commitment_number` != 1 and you haven't sent `commitment_signed`:
- Your peer thinks you've sent a `commitment_signed` that you haven't
- You've forgotten that you sent a `commitment_signed`

`next_commitment_number` > your last `commitment_signed` + 1 
- They say that they have received a commitment that's further in the
  future than you believe yourself to be.

Q: what can we do here?

Revocation Number - Happy case
==============================
`next_revocation_number` = your last `revoke_and_ack`:
- They have not received your `revoke_and_ack`, retransmit

`next_revocation_number` = your last `revoke_and_ack` + 1:
- You are on the same page re the last revocation you've signed

Revocation Number - Bad case
============================
`next_revocation_number` != 0 and you haven't sent `revoke_and_ack`:
- Your peer thinks you have sent a `commitment_signed` that you haven't
- You've forgotten that you sent a `revoke_and_ack`

`next_revocation_number` > your last `revoke_and_ack` +1
- They say that you have revoked a commitment that's further in the 
  future than you believe yourself to be

If `your_last_per_commitment_secret` is valid for 
`next_revocation_number` - 1 then you know that you have lost state, 
because they have a valid secret (which means that you _did_ send that
revocation*).

* Or they compromised your secret store, but of if that's the case 
they can already steal all your $$ so it's moot.

## Questions

When would we have used a payment_preimage before we've received 
`commitment_signed`? There's a specific note in the spec about this.
