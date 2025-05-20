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

## Privacy Concerns

There is currently some debate over whether we should report payments
with millisecond granularity, or update the encoding to represent
100ms "chunks".

Concern:

Why can't this be enforced on sender-side?
- If we give nodes ms-granularity, they'll always be able to cheat and
  set a different value
- Technically this is still possible by flipping a high bit in the 
  payment hold time field, but it would require that senders upgrade
  to understand it.

How to timing attacks work?

[Counting Down Thunder](https://arxiv.org/pdf/2006.12143)
- "the off-path unobservability of payments could potentially be subject to a deanonymization attack run by a global passive adversary"
  Q: How?
- Attackers may use amount, CLTV and timing analysis to compromise:
  - On path sender/receiver privacy: forwarders don't know sender/receiver
  - Receiver's sender privacy: the receiver doesn't know who paid
- To probe latencies of nodes in the graph:
  - Construct payments that are bound to fail at different hops and
    measure latency (can isolate one link at a time)
  - Track time between `update_add_htlc` and `update_fail_htlc`
- Destination deanonymization:
  - Record the time between receiving `update_add_htlc` and 
    `update_fulfill_htlc`
  - This gives a timing estimate relating to the recipient
- Source deanonymization:
  - Requires failing back a HTLC, and again being chosen as the
    retry path (which happens instantly)
  - Observe time between two `update_add_htlc`
  - This gives a timing estimate relative to the sender
- It's tempting to blame CLTVs, but removing this fingerprint entirely
  only decreased precision and recall by 2-3%
  (we really can never remove amount, so it's not worth thinking about)

Takeaways:
- We should have a more significant delay between retires if we don't
  already
- LND should penalize channel update related errors if it doesn't 
  already
- Q: What was the accuracy of latency probing in the paper?
  IE, how much worse are we making it by surfacing this information
- Q: How does batching help this, doesn't it just get counted towards
  more jittery latency?

Intuition: if you add enough latency that it may be inferred to be an
additional hop, then you break this heuristic. Therefore, a reasonable
mean latency to be inserted is the amount of one hop; if each hop does
this then you're getting quite a lot of ambiguity.

Sender De-Anonymization:
- Introduce a reasonable delay between retries
- Sender gets to choose whether they want to be patient for privacy

Receiver De-Anonymization:
- Receiver can choose to introduce latency on their received payments
- Attacker measures round trip time, so it doesn't matter where it is
  introduced
- Senders in an attributable errors world can give privacy conscious
  recipients some grace:
  - They still get good information about the speed of forwarding nodes
  - In future, we'll have a "slow payment" signal for jamming anyway

In this model, a sender can identify whether a recipient is including
a delay, but then they already know the receiving node anyway.

Assuming RTT:
* 20 ms: good
* 50 ms: mid
* 100 ms: regular

We need 3x RTT to add/remove HTLCs, so we're seldom going to have a value
that's less than 150ms in practice. This doesn't really leave much time
for batching?

Problem:
- We're going to start surfacing hold times
- People might start using them in pathfinding
- We don't want to disincentivise delays (good for privacy)

[Censorship despite communication](https://drops.dagstuhl.de/storage/00lipics/lipics-vol316-aft2024/LIPIcs.AFT.2024.12/LIPIcs.AFT.2024.12.pdf)
- Network level censorship makes LN susceptible to censorship
- Network level actor can examine the length of messages and interfere
- This can be used to infer sender/receiver information

===================
Response on the PR:
===================

Writing this out in lay-carla's words:

### Problem
* We're going to start surfacing hold times using attribution data
* It's probable that people start to use these hold times as input to
  pathfinding algorithms to get the fastest route possible
* This would dis-incentivise adding forward delays, which help privacy

Per [@tnull's paper](https://arxiv.org/pdf/2006.12143):
* An on-path adversary can observe the time between `update_add_htlc`
  and `update_fulfill_htlc` to get the latency between itself and the
  recipient
* It can then compare to known latencies of possible paths (using CLTV +
  payment amount to eliminate some options) to find highest probability
  recipient.
* If the hops along the route add sufficient delay to look like another
  hop, this estimation will be off (give or take some nuance I'm ignoring)

There's also a trick for deanonymizing senders, but that's fixable by
adding a reasonably wait before retrying or using a distinct retry 
path.

* TODO: network level adversary

### Solution

If we change the encoding of our `hold_time` so that the sender can't
distinguish values < `X`, then we do not dis-incentivise privacy 
protecting forwarding delays.

`X = reasonable latency for a hop + delay`

For example, if `X=100` then we express our `hold_time` as blocks of
`100ms` (1 = 100ms, 2 = 200ms etc).

However, this comes at the cost of a sending node not being able to 
identify incredibly fast forwarding nodes because they're lumped in
with nodes that took up to `X` to complete.

#### What is a reasonable value?

Per self-reports in the last spec meeting, delay times are:
* Acinq: 0
* LND: 50ms or 10 HTLCs
* LDK: 50ms

Assuming a median network RTT of 50ms we have a minimum hop latency of
150ms (3 RTT required to add/remove a HTLC). With 2-3 hops per route,
we'd expect to approximately hit our requirement that delays add 
sufficient latency to help with deanonymization (3*50 ms = 150 ms = 3x RTT).

Notably, [TOR onion servers RTT](https://metrics.torproject.org/onionperf-latencies.html)
media sits around 500ms. So realistically we're not really accounting
for them here at all.

#### What if we need a bigger number in future?

In the case of an in-path attacker, they're measuring the route trip
from the receiving node so I think that it would be reasonable to add
any additional delay on the receiver's side:
* Timing attacks use round trip time, so it doesn't matter where this
  delay is introduced.
* The receiver has the best incentive to protect their own privacy and
  delay the payment.
* The sender still gets a view of which forwarding nodes are fast, and
  can ignore the recipient's longer hold
 
So TL;DR, I think that we should just plug in whatever value of `X`
we think is informed/reasonable and decouple this change from the
conversation of whether we need to increase this delay today because:
- We can do it on the receiver to address an in-path attacker
- Network level adversary is so much more involved; IMO it is sufficient
  to leave this case no worse than it is today

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
