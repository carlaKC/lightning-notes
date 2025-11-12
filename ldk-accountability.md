# Accountability API in LDK

Based on ldk-2855, it looks like we'd want a completely separate trait
to handle DOS protections (do no overload htlc interception, which is
a more user-facing concern). Consensus also appears to be that we don't
want to overload user-facing events with a solution which should be
internal to LDK (with the option to override it).

## Design Considerations

*Should this trait be implemented as a blocking handler or with queuing
and callbacks?*
- We will be calling this trait at a performance-critical point in the
  codebase, so we can't tolerate any blocking operations.
- There is always a risk of externally implementations of the trait
  accidentally blocking.
- The queue itself if not at risk of denial of service because we only
  allow 483 HTLCs per channel.
- Queuing will be fast, and we can batch resolution to speed up
  operation.

-> Decision: this should be implemented with a queue and callback, like
   htlc interception is currently implemented in LDK.

*Should this trait be called before or after the existing (and future
general) interception API?*
- We should allow user-controlled interception APIs the opportunity to
  see all traffic, so that they can take custom actions on it.
  -> For example, intercepting a phantom receive will result in a
     custom action that results in our not needing to "forward" at all.
- We should be wary of DOS interception interfering with custom actions
  that the end user might want to take.
  -> For example, we intercept a htlc to make a JIT channel, only to
     fail it back due to DOS protections.
  - Options:
    - Add an `override_dos` option in regular interception
    - Implement awareness in our default impl (if possible)
    - Require custom implementation of DOS trait when taking custom
      actions

-> Decision: implement DOS trait after regular interception, and add
   awareness for custom use cases like JIT channels in default impl.

*How should we handle the case where an implementation hangs/does not
give us a response for a HTLC?*
- In the same way as we handle `pending_intercepted_htlcs` in
  `do_chain_event`; if we have not heard back from the impl then we
  fail it back.

## Implementation

We define a trait that can implement custom DOS protections:
```
pub trait ResourceManager {
  allocate_htlc_resources(Vec<u64, PendingAddHTLCInfo>) -> Result<(), ()>
}
```

We keep track of the following states for resource checks:
```
enum ResourceAllocationStatus {
  AwaitingNotification, // We've queued it, but not asked about it
  AwaitingResponse, // We've asked the trait about it, but not heard
  Allocated(Allocation), // We've got a response for it
}
```
Channel manager has access to this trait, and tracks htlcs that are
awaiting a response from the `ResourceManager`:
```
ChannelManager <R: ResourceManager> {
  resource_manager: R,

  pending_resource_allocation: Mutex<HashMap<u64, Vec<(HTLCForwardInfo, ResourceAllocationStatus)>>
}
```

We will queue htlcs that need a resource decision in `forward_htlcs`:
- If not intercepting a htlc:
  - If not present in `pending_resource_allocation`:
    - Add to `pending_resource_allocation` with `AwaitingNotification`
  - Loop through all `pending_resource_allocation`, failing or forwarding
    any htlcs that have `Allocated` verdict set in our map.

These htlcs will periodically be passed to the trait implementation
by adding a task to `BackgroundProcessor::start()` which calls
`ChannelManager::process_pending_resource_allocations`:
- Lock `pending_resource_allocation`
- Pass all htlcs that are `AwaitingNotification` to the trait

If we don't want to poll here, we can use a `Notifier`?

There will be a `allocate_resources` handler on `ChannelManager` which
accepts responses from the trait implementation:
- Lock `pending_resource_allocation`
  - Lookup htlc, fail if not `AwaitingResponse`
  - Set `Allocated` in map

As is the case with `pending_intercepted_htlcs`, we'll add handling
in `do_chain_event` that makes sure that we cancel these back if we
don't hear back in time.

### Persistence

- `forward_htlcs` are written in `ChannelManager::write`
- `pending_intercepted_htlcs` are written in `ChannelManager::write`

We will have to return a `NotifyOption` in our processing of pending
resource allocations so that we can store these htlcs. On restart,
we'll play all htlcs awaiting notification to the manager.

### Default Implementation

To start with, we'll just have experimental signaling support that
copies the incoming value to the outgoing value. This will be trivial
to implement.

To make sure that this API will work, take a look at what implementing
reputation in LDK would look like (internally and externally).

#### Internally

We have `per_peer_state` / `channel_by_id` and we only care about
`FundedChannel` state (since others are pre-htlcs), which keeps all
of its data in `ChannelContext`:

When we are notified of a htlc that we need to assign resources for,
we are given the following information:
```
pub(super) struct PendingAddHTLCInfo {
	pub(super) forward_info: PendingHTLCInfo {
	  pub routing: Forward {
		onion_packet: msgs::OnionPacket,
		short_channel_id: u64, // This should be NonZero<u64> eventually when we bump MSRV
		blinded: Option<BlindedForward>,
		incoming_cltv_expiry: Option<u32>,
		hold_htlc: Option<()>,
	  },
	  pub incoming_shared_secret: [u8; 32],
	  pub payment_hash: PaymentHash,
	  pub incoming_amt_msat: Option<u64>,
	  pub outgoing_amt_msat: u64,
	  pub outgoing_cltv_value: u32,
	  pub skimmed_fee_msat: Option<u64>,
    },
	prev_outbound_scid_alias: u64,
	prev_htlc_id: u64,
	prev_counterparty_node_id: PublicKey,
	prev_channel_id: ChannelId,
	prev_funding_outpoint: OutPoint,
	prev_user_channel_id: u128,
}

```
-> We can identify the HTLC uniquely on the incoming channel (scid/id)
-> We can calculate the fee
-> We know the incoming cltv

We need to track the following internal state:
```
channels HashMap<u64, ResourceState>
```

Where we maintain the following additional state:
```
ResourceState {
  incoming_revenue: u64,
  outgoing_reputation: u64,
  in_flight: HashMap<u64, BucketAllocation> 
}
```

To keep our state in sync with the state of channel manager:
- For each of the htlcs that we're adding:
  - For each outgoing channel:
    - Iterate through `pending_outbound_htlcs` and calculate risk
    - [ ] We don't have htlc fee here
  - For each incoming channel:
    - Iterate through `pending_inbound_htlcs` and delete any `in_flight`
      that we're tracking that are no longer present.

Since this is an internal implementation, we can rely on the locking
being properly handled and not blocking for too long.

TODO: we also need to be able to track reputation/outgoing reputation
values, which means we need to know when things are settled/failed. I
think this would always be okay because we hit this at the process
forwards stage (where settles/fails are piped) but might need to worry
about consistency a bit there.

#### Externally

- We get a `allocate_htlc_resources` call, which gives us our new set
  of htlcs for each channel (+ all info we need, like expiry).
- `PaymentForwarded` informs us if the forward was successful.
- `HTLCHandlingFailed` informs us if the forward failed.

Missing:
- [ ] We don't have HTLC id on any of our events, which makes it
      difficult to identify them
- [ ] We don't have an easy way to get an idea of our current state
      when we start up?

tl;dr: doable, but we'd need to surface some new information and surface
information about our history to the trait.

## Setting Accountability Signal

Where will we need to set/save this signal?

`UpdateAddHTLC`:
- So that we can receive/send the signal in P2P messages
- We use `impl_writeable_msg!` to persist this
  * We can possibly skip this as follows: `(1, accountable, (legacy, u8, || None::<u8>))`

When we call `forward_htlcs`:
- `PendingHTLCInfo` contains all the information about the current htlc
  -> We should add an `accountable` signal to `PendingHTLCInfo`

Where do we construct these?
- `get_pending_htlc_info`: we have the incoming `UpdateAddHTLC` on hand
- `forward_intercepted_htlc`: from values in `PendingAddHTLCInfo`
  -> We should add `accountable` to `PendingAddHTLCInfo` 
- `create_fwd_pending_htlc_info`: we have the incoming `UpdateAddHTLC`
- `create_recv_pending_htlc_info`: we appear to mostly have the
  `UpdateAddHTLC` around at call sites, even though we don't provide
  it here (we use components).
- `peel_payment_onion`: we have the incoming `UpdateAddHTLC`

Q; can we add this signal to `PendingHTLCInfo`'s `routing` which is
`PendingHTLCRouting`?
-> No, this is created directly from our onion packet


- We're going to call our trait in `forward_htlcs` to get our outgoing
  accountable status.
- We can get the signal from `source` here, even though we could
  _technically_ set it later inside of `channel`
- This value will be put into `forward_htlcs`:
  - `HTLCForwardInfo::AddHTLC`
    - Add the signal to this variant
- We'll process the htlcs in `forward_htlcs` in 
  `process_pending_htlc_forwards`:
  - We'll have our signal already here, because we called a trait
  - We put in a holding cell as `HTLCUpdateAwaitingAck::AddHTLC` 
    -> We will need to add the signal here
  - We then graduate to `pending_outbound_htlcs` as `OutboundHTLCOutput`
    -> We will need to add the signal here
    - From here, we'll make our `UpdateAddHTLC`

Places where we need to add `accountable`:
- [x] `UpdateAddHTLC`
- [x] `PendingHTLCInfo`
- [-] `PendingAddHTLCInfo`: we already have `PendingHTLCInfo`,
      and we need this information because this is what we store in
      `forward_htlcs`
- [ ] HTLCForwardInfo
- [x] `HTLCUpdateAwaitingAck::AddHTLC`
- [x] `OutboundHTLCOutput` (can add to UpdateAddHTLC)

Figuring out where to switch the HTLC signal:
- We are going to make the call in `forward_htlcs`
- We need to have the incoming signal on hand for when we add
  interception:
  - That is contained in: `PendingHTLCInfo`
- We also need to be able to set the `outgoing_accountable` from
  `forward_htlcs` and have it propagate through to the outgoing link:
  - This information is stored in `HTLCForwardInfo::AddHTLC`
    - This contains `PendingAddHTLCInfo`:
      - This contains `forward_info` which is a `PendingHTLCInfo`
        (already has the incoming signal)

Q: Where should we put it?
- `forward_info`: this is passed in, we don't create it
  We'd need to set it really early on as `None` and then replace it
  with our outgoing signal in `forward_htlcs`

Option 1:
- Put `outgoing_accountable` in `PendingHTLCInfo`
- Issue here is that we create this info pretty early on, and then
  have to mutate it later.

Option 2:
- Turns out this is actually going to an issue if we put it in
  `PendingAddHTLCInfo` as well, it's just shorter-lived problem.
- It also seems a bit disjunct to have `incoming` in `forward_info`
  and then `outgoing` on the higher layer (now that I've written it)

## Notes for an issue/PR

Things to mention in the issue:
- I'd like to implement blip04 (which also needs an update)
- Persistence options:
  - Explicitly do not write it (because we can tolerate dropping)
  - Just write it as an odd TLV and be happy to stop when deprecated
- Is it okay to change the API of `send_htlc_and_commit`?
  I'm not sure how much this is are used? We could also hard-set a
  zero value

Q: Should we put this in `PendingHTLCRouting`?
- This is kept in `PendingAddHTLCInfo`, which is what we will
  store in our `forward_htlcs`
- If we set our outgoing in `process_pending_forwards` then we don't
  have to set anything!

```
A = process_forward_htlcs (called by internal_process_pending_htlc_forwards)
B = forward_htlcs (where we push htlc to ChannelManager.forward_hlcs) 
```
## Design Runthrough

Priorities?
- Performance

Questions to answer:
- Should it be part of general interception?
- Should it be in `forward_htlcs` or `process_forward_htlcs`?
  (plus, before or after custom interception?)
- How do we handle asynchronous responses?  
- How should we handle non-response from API?
- How should we handle interceptor errors?
- What do we need to persist?

Notes:
- Consensus from issues is that we want to have a totally separate API
  for this
- Seemed pretty clear that we don't want to use events for this, we
  want to have a trait that people can sub out if they want to.
- Where should we put it:
  - `process_forward_htlcs`:
    - Locks: `per_peer_state`
    - Pro: outgoing channel is chosen, important for reputation algo
    - Con: logic seems a little out of place here (ie interception elsewhere)
  - `forward_htlcs`:
    - Locks: `forward_htlcs`, just map is locked
    - Con: outgoing channel can change
    - Pro: co-located with other interception code
- Inline handler vs async callback:
  - Inline: if we're only going to surface internally
  - Callback: if we want to allow external impls
- Non-response?
  - Inline: "just don't"? 
  - Callback: do_chain_event will time out htlcs that are expiring

Q: [KC] double check locking on forward_htlcs
Q: [KC] could we pick the optimal_channel sooner?
-> If yes, maybe do this in forward_htlcs (pending above lock check)
Q: [M] Where would be a good point to notify reputaiton manager of
   settle/fail
Q: [M] restarts and replays

- `send_htlc`: accepts an `accountable`, called by:
  - `queue_add_htlc`:
  - `free_holding_cell_htlcs`
  - `send_htlc_and_commit`:

Q: Should we surface on `send_htlc_and_commit` or just set to `Some(0)`?
- It _seems_ like this is only really used internally anyway, so it's
  okay to just use a bool.

Q: Should this be a `Option` or a `bool`?
- On the "external" facing API things, this makes sense as a bool
  - Somebody absolutely will set `Some(1)` meaning for that to be true
    but our funny `Some(7)` will fuckem
- Just have the `Option` where we've got the persistence layer that
  we can't handle., conversion is really easy.

[Forwards] `channelmanager::process_forward_htlcs`:
- Calls `queue_add_htlc` -> `send_htlc`

[Forwards] `channel::free_holding_cell_htlcs`:
- Calls `send_htlc` using `accountable` from `HTLCUpdateAwaitingAck::AddHTLC`

`channelmanager::send_payment_along_path`
- Calls `channel.send_htlc_and_commit` -> `send_htlc`
