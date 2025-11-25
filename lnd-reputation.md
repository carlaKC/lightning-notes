# LND Implementation Notes

For each channel we track, we need to track the following information:
- Revenue from incoming forwards
- Reputation from outgoing forwards
- Current "bucket" allocation

Our outgoing link handles the packet in `handleDownstreamUpdateAdd`:
- At this point we do have all the information that we want for
  reputation decisions (fee, accountable signal, timeout)
- We also have the HTLC that we want to send out on our hands, so we
  could set an outgoing accountable signal.
- However, we'd need to "reach back" to get the revenue information for
  the incoming channel, so perhaps this isn't the best spot for it.

We originally create our packet in `processRemoteAdds` on the incoming
link, and then sent via `ForwardPackets` to the switch.

`handlePacketForward` seems like a good spot - we're in the switch,
so we have access to our channel links. We also do a bunch of rejecting
of things in `handlePacketAdd`.

Note: could also have handling for settle/fails here, because we've got
everything on hand.

Q: How should we keep data consistent?
- `channellink` has an underlying `LightningChannel`
  - We have a `StateSnapshot` which returns all the `Htlcs` on our
    commitment (amt / expiry / accountable known)

NB: we don't have fees available here, but we could get the ID to help
look up somewhere else?

We could add to `CircuitMap`:
- `ListByOutgoing(scid)` returns the indexes of all pending htlcs and
  their incoming/outgoing amount (which gives us fees).

What if we had a reputation tracker that caches the full cost of each
of the incoming htlcs by `CircuitKey`

TODO: reputation isn't decaying average

## Restarts

When we restart we will need to know:
- Channel Revenue (start_time / avg { value, last_updated})
- Outgoing reputation (avg { value, last_updated })
- Extra in flight data {added_at, added_height}

Q: where can we re-load HTLC info
- We create a new manager in server.go, where we call switch.New
- If we use the combination of the circuit map (new `ListCircuits`
  call) and `FetchAllOpenChannels`), we should have all of the htlcs
  info that we need:
  - Incoming/Outgoing link: circuit map
  - Fee: circuit map (incoming/outgoing)
  - Incoming Expiry: incoming channel's `HTLC`
  - Accountable: outgoing channel's `HTLC`
- We can reload as follows:
  - Pull all circuits that have outgoing circuits
  - Pull all channels, add htlc info from channel

- Things that won't be available to us:
  - Time htlc was added: would need to add this to payment circuit
  - Height htlc was added: accept some loss here (just calc from addedTs)

-> It would be nice if a timestamp was available to use somewhere else
   in the codebase, perhaps added for attributable failures?
It's added in `ErrorEncryptor`, which is kind of a strange spot to have
it. We probably want to add it somewhere else that's more general purpose.

Steps for restart:
- [x] Move circuit map creation outside of switch
- [x] Add implementation to circuitmap to provide current circuits
  [ ] Possibly change this back to pointers because we're going to
     just copy at the caller base anyway.
- [ ] Provide circuitmap list to reputation manager + `FetchAllOpenChannels`,
  and reconcile information to build reputation

Next up!
- We'll need to store timestamps on htlcs somewhere to be able to
  restore

Q: Can we have this on a per-commitment level?
- No, because HTLCs can be added a few commitments back so we're going
  run into some nasties trying to figure out when.

Q: Is it appropriate to store timestamps by HTLC in `channel`?
- With the HTLC _might_ be the idea, but the persistence of htlcs is
  a fucking disaster.
