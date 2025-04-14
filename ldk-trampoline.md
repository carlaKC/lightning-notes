# Trampoline Routing

Top level notes for reviewing trampoline routing in LDK.

Top level tracker: https://github.com/lightningdevkit/rust-lightning/issues/2299

Mechanics: https://github.com/lightning/bolts/pull/829
Onion Format: https://github.com/lightning/bolts/pull/836

## First Readthrough

# 829
- How to nodes that are doing trampoline sync gossip? Should we have
  some allowances in new gossip to account for this
  - Some vague comments are included at the end of the doc
- Why does the trampoline need to have payment secret / where does it
  come from?
  - The intermediate payment secrets are randomly generated

# 836
- Why do we have an outgoing node id in the onion packet?
  - This tell us where the next trampoline is (we don't have SCID)
- How do decoding nodes know how big the variable onion packet is?
  - It's inside of a TLV, so we know total length and all other values
    are fixed length so we can infer the length of it
- Where's the eph pubkey for the trampoline?
  - In `trampoline_onion_packet`
- Is MAY use recipient features strong enough?
