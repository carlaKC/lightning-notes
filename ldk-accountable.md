# Experimental Accountable API 

- Add the signal
- Add a trait that just propagates the signal by default

## Add experiemntal accountable signal htlcs

- In `channel::read` we get all our `pending_outbound_htlcs`
- We set a few fields there to none (like hold time)
  - These need to be independently serialized, because they're not
    inline with the other htlc information.
- For this impl, we don't bother writing them (?)

## Propagate Experimental Accountable Signal



Claude says that the path is: 
  Key Propagation Path:
  Incoming: UpdateAddHTLC.accountable
      ↓
  Store in: InboundHTLCOutput.accountable
      ↓
  When forwarding: HTLCPreviousHopData.accountable
      ↓
  Create outbound: OutboundHTLCOutput.accountable
      ↓
  Send: UpdateAddHTLC.accountable

  The signal needs to flow through the entire payment
  path, being preserved as it moves from inbound →
  storage → forwarding logic → outbound.


