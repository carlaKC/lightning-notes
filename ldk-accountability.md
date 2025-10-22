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

TODO: finish tracking what we'd need here.

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
