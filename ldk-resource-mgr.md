# Resource Manager Early Review

Thoughts on to-dos:

> Need to rethink how we allocate slots based on size of channel

Let's ask Clara to take a look and give general advice on this:
- How to allocate slots based on total slots in channel
- How to allocate liquidity in congestion bucket when it's quite small

> In read-only mode, the resource manager will add the HTLC and say
  we should fail it, but the channel manager will just log that and
  forward it anyway.

Can we make the resource manager a trait and then wrap one impl in
another? That way the read only one can just act as-if it failed it
immediately, and ignore any resolutions that come in.

> Bucket resizing?

Yuck, good question. We definitely need to be able to adjust based on
splices that happen. That seems messy - what if we get smaller and we're
now over our limit?
- They don't change _right now_
- Check that SCID doesn't change, just make sure we call with the alias

> This won't work for `nostd` because we only have block times.

Does LDK server/node use `std`? If so, I think we're fine. We should
just turn the manager off if we're in `nostd`. Perhaps gate the nop
impl on `nostd`?

## High Level Review

1200 lines for reputation manager
1000 lines of tests

Interactions with `channel_manager` look relatively clean.

Q: Why doesn't the trait also extend `Readable`?

Q: How does channelmanager handle readable/writable elsewhere? 

Q: Why do we need entropy source to be clone?

Q: Why do we map to accounatble = true when we would otherwise have
   failed?

Q: It would be good to be able to log a bit more information about the
   HTLC + reputation scores along with our reading.

Q: Why do we resolve in `claim_funds_internal` vs `process_forward_htlcs`?
- One is forwarding and the other is forwarding and receiving.

Q: We should generally use off TLVs to remain backwards compat (applies
   in a few places).

nit: Can remove default comments in `ResourceManagerConfig` and put them
     on the `Default` impl where they need explaining.

Q: Can we keep `opportunity_cost` as `i64` during calculations or do
   we lose necessary precision to be linear?
- I think that we need this to be `f64` but double check

Q: Should definitely fix up locking!
- Note in commit why we don't want a `RWLock` (this was a wrong thought
  I had).

Q: Remove `incoming_channel` and `htlc_id` from `PendingHTLC`

Q: Should we track in flight risk as a single number? Would that save
   us much?

Q: Use a hashset for `available_slots` and `congestion_htlcs_from`

Q: Pass time in from caller, we can possibly fix for multiple htlcs?
   I think that we already have it available for most HTLCs?

Discuss:
- Reviewing on fork vs reviewing on LDK (?)
