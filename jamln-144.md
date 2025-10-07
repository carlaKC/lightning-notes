# Slow Jamming Attack Impl

- `SlowJam`:
  - Has an attacker sender/receiver
  - Has a map of all the target node's channels: we just use this to
    check that we have a channel with the target as the attacker.
  - Takes a `ChannelJammer` (which allows us to jam general / congested)
  - Has a network graph showing topology
  - Tracks payment hashes that are used for channel jamming!

- `run_attack`:
  - Build reputation using the attacking nodes:
    - Set our route to be: attacker sender -> target -> attacker receiver
    - We set our amount to be the liquidity of the channel that we're
      using
    - `build_reputation`:
      - Build our route with a 1000 msat htlc
      - Add on fee that we need to get reputation for all htlcs 
      - Send and track payment, check that we have enough rep after
  - Once we've build our reputation, we general + congestion jam
    the target channel (target pubkey, target channel)
    - Get the target node
    - Get its channel
    - Set its incoming direction to jammed
  - `slow_jam_channel`:
    - Get the sending attacker node
    - Create path: sender attacker -> channel to jam -> target -> receiver
    - Hard code the amount that we need to jam
    - We `send_to_route` then add `jamming_payments`
  - Once we've done our jamming, we test whether an honest receiver can
    get a payment

- Implementation of jamming trait:
  - We hold for the `incoming_expiry_height`'s block time (ok because
    we have a zero height).
  - We take no actions on forwards

Q: Is this zero?
[ ] Check logs!

Q: Shouldn't we be doing slot jamming and it would be cheaper?

Q: what does `santiy_check_node` do? What's special about 69
