# Attributable Errors

## Motivation

Source-based lightning payments use the errors that they receive from
failed payments to inform their next attempt at making a payment through
the graph by penalizing the failing node in their internal scoring.

Today's errors are insufficient in a few ways, per [1]:
- Successful payments that take a long time to resolve can't be 
  attributed to any individual node along the route.
- Malicious nodes can corrupt the failure message, which prevents the
  sender from pinpointing the source of failure.

## Current Error Format 

The node that sends an error can't create an onion packet like the 
original sender did, because they don't know the full route for the 
payment (or the ephemeral private key used for the original onion).

When forwarding payments, intermediate hops store the shared secret
from the original forward which is used to obfuscate errors [2]. This 
is use to generate two keys [3] for error processing:
- `um`: used to produce a HMAC over the error message, ensuring its
  authenticity and integrity.
- `anmag`: used to produce a pseudo-random stream of bytes, which is 
  XOR-ed with the packet to obfuscate it.

When a node fails a payment it will:
- Create a failure message (2 byte type plus data fields), pad its 
  length and add a HMAC over the contents. 
- Obfuscate the error and send it back down the route as part of an
  `update_fail_htlc`.

Every node along the path will repeat the obfuscation step with their
own `anmag` key, XORing the contents of the packet. Once the original
sending node receives the error, they can generate the `um` and `anmag`
keys for each hop in the route and iteratively de-obfuscate the packet,
reversing each hop's XOR with their own. The sender is detected by 
matching their expected HMAC with the one provided in the error.

What's wrong with this?

Anybody along the route can just corrupt the failure message, and the
sender won't be able to determine who it was.

## How do attributable errors work?

Example: A - B - C - D - ...
- Sent by A
- Fails at D

### Erroring Node (D)

The error message that is created by the erring node is updated to 
include two additional fields:
- `payloads`: 20 payloads containing additional error information [4]
- `hmacs`: 210 x 4 byte hmacs  TODO

The failing node will insert their data in the first payload and zero
out the rest:

`[1/5s][00000][00000][00000][00000][00000][00000][00000][00000][00000][00000][00000][00000][00000][00000][00000][00000][00000][00000][00000]`

As this message is propagated back along the route the erring node's 
payload will be shifted to the right as other nodes add their payloads.
The node wants to be able to provide a HMAC to the original node that 
will properly cover its location in the route, and it doesn't know how
far it is in the route, so it'll generate a series of HMACs for each of
its possible locations in the route.

* Distance 0: `H0_0=HMAC([1/5s])`
* Distance 1: `H0_1=HMAC([1/5s][00000])`
* Distance 2: `H0_2=HMAC([1/5s][00000][00000])`
...
* Distance 19: `H0_19=HMAC([1/5s][00000][00000]...[00000])`

It'll just concatenate all of these HMACs:
* `H0_0 | H0_1 | H0_2 |... | H0_19`

### Intermediate node (C)

When the next node receives the error, they will shift all payloads to
the right and add their own in the leading position [5]:

`[0/2s][1/5s]...[00000]`

It will then likewise create HMACs for each one of its possible 
locations in the route. Note that this time, the HMACs will cover some
of the previous node's values:

* Distance 0: `H1_0=HMAC([0/2s])`
* Distance 1: `H1_1=HMAC([0/2s][1/5s])`
...
* Distance 19: `H1_19=HMAC([0/2s][1/5s][00000]...[00000])`

The first HMAC that each node adds to its set of 20 HMACs is assuming 
that it is next to the sending node. Given that an intermediate node is
receiving the error, not the sender, it knows that this cannot be the 
case. This means that it's safe for the intermediate node to remove the
first HMAC for each set of 20.

It has received: 
* `H0_0 | H0_1 | ... | H0_19`

It will trim this to: 
* ` H0_1 | ... | H0_19`

Note that this has trimmed (H0_0) the HMAC that covers the case where 
the erroring node is right next to the sender (correctly so, as this 
node isn't next to the sender).

It will then add its own HMACs in front:
`H1_0 | H1_1 | ... | H1_19 | H0_1 | ... | H0_19`

### Intermediate node (B)

Repeating this once more for another intermediate node, shift and add 
payload:

`[0/3s][0/2s][1/5s]...[00000]`

Now, generate HMACs for each possible location of the node (as above):
* Distance 0: `H2_0=HMAC([0/3s])`
* Distance 1: `H2_1=HMAC([0/3s][0/2s])`
...
* Distance 19: `H2_19=HMAC([0/3s][0/2s][1/5s]...[00000])`

This node has received the following HMAC chain:
* `H1_0 | H1_1 | ... | H1_19 | H0_1 | ... | H0_19`

It can proceed to prune the _first_ HMAC from each node's set:
* `H1_1 | ... | H1_19 | H0_2 | ... | H0_19`

Note that this has trimmed the HMAC that covered the case where the
erroring node is one hop from the sender (`H0_1`) and the case where the 
first intermediate node is next to the sender (`H1_0`). Neither of these 
cases are possible when we're at the second intermediate node (though it 
doesn't know its actual location in the route).

It'll append its HMACs to the chain and send the error backwards:
* `H2_0 | H2_1 | ... | H2_19 | H1_1 | ... | H1_19 | H0_2 | ... | H0_19`

Note that for each of these steps, we're still XOR-ing the full failure
with our `anmag` key. This means that we're shifting obfuscated values
when we modify the payloads and HMACs - this is fine, because we know
the length of each field.

### Receiving Node (A)

When the original sending node receives the error, it will get the
following data:

Payload: `[0/3s][0/2s][1/5s]...[00000]`


HMACs: `H2_0 | H2_1 | ... | H2_19 | H1_1 | ... | H1_19 | H0_2 | ... | H0_19`

It'll do the usual step of XOR-ing the payload for each hop, but will
include an additional step of checking HMACs against payloads. The 
sending node knows which position each node is in, so it can go ahead
and pull out the correct HMACs for each hop and iteratively validate
them against the appropriate message.

For intermediate node B:
------------------------
Payload: `[0/3s][0/2s][1/5s]...[00000]`


HMACs: `H2_0 | H2_1 | ... | H2_19 | H1_1 | ... | H1_19 | H0_2 | ... | H0_19`

- Get their payload for distance 0: `[0/3s]`
- Verify HMAC for distance 0 from source: `H2_0=HMAC(0/3s)]`
- Trim remaining 19 HMACs from the node.
- Left shift payloads by one.

For intermediate node C:
------------------------
Payload: `[0/2s][1/5s]...[00000]`


HMACs: `H1_1 | ... | H1_19 | H0_2 | ... | H0_19`

- Get their payload for distance 1: `[0/2s][1/5s]` 
- Verify HMAC for distance 1 from source: `H1_1=HMAC([0/2s][1/5s])`
- Trim remaining 18 HMACs from node.
- Left shift payloads by one.

A few things are of note here: 
- The further we get from the error source, the fewer HMACs we have to
  trim, because they would have been trimmed along the way.
- When C produced their payload for being distance 1 from the source,
  D's payload was already in the payload array and would have been 
  covered by the HMAC.

For erroring node D:
--------------------
Payload: `[1/5s]...[00000]`


HMACs: `H0_2 | ... | H0_19`

- Get their payload for distance 2: `[1/5s][00000][00000]`
- Verify HMAC for distance 2 from source: `H0_2=HMAC([1/5s][00000][00000])`
- Discard remaining payloads and HMACs because valid payload with source
  of one is found.

## How do we upgrade?

This is a full path upgrade, because every node on the route would need
to know whether to use this new error format or not. Nodes along the
route need to understand which type of error to send, and only the
sender knows whether the full path supports attributable errors 
(assuming we have a feature bit for this. This can be done by signaling
in the onion, or in `update_add_htlc`.

## What doesn't this solve?

This doesn't solve the issue of attributing blame between a pair of 
nodes when payments fail with `update_malformed_htlc`, because you have
no way of knowing whether the sending or receiving node had an issue.
You assume that you've formed a correct HTLC, so something must have
gone wrong with one of those nodes. A malicious node can always send 
back a malformed error when they want to be fishy, even with new errors.
The solution is just to penalize the pair in your pathfinding scoring.

## Footnotes
[1] https://lists.linuxfoundation.org/pipermail/lightning-dev/2019-June/002015.html


[2] Each node in the route receives an onion packet with a `public_key` 
    and an encrypted set of `hop_payloads`. Elliptic-curve Diffie-Hellman
    is used to produce a shared secret using this ephemeral key and the 
    node's private key (the sending node did so with the private key of
    the ephemeral key and the forwarding node's public key). The point
    output by ECDH is serialized in its compressed format and SHA256ed
    to produce a 32 byte shared secret value.


[3] Keys are generated as HMACs, using the key type (`um`=`0x756d`, 
    `anmag`=``) as the HMAC-key and the 32 byte shared secret as the 
    message. 


[4] To keep the size of these errors reasonable, we assume that our 
    maximum path length is 20. We can't just add a single payload for
    each hop because we trivially expose our location in the path this
    way.


[5] Note that there's no incentive for intermediate hops to lie about
    being the source of the error, because they'll just be blamed for
    the failure (they _want_ the next node to be attributed).

## Questions
- Why do we use HMACs to generate our keys for encryption?
- Since we're shifting HMACS and payloads known to be the last X bytes
  of the error message, we can never add any TLVs? That would break the
  "last X" that we're currently relying on - prehaps move them to the
  front.
- The current proposal includes duration held, but I think that we need
  include add_sent and resolve_recieved timestamps for each node - the
  matter of figuring out who lied isn't simple.
