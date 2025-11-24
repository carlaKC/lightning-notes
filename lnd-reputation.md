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

Q: what about restarts?

Notes:
- Have a struct that gets passed in
- I don't think we need to store incoming expiry height (just blocks held?)
