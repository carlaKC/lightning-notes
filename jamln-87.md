# Fix duplicate slot bug

`candidate_slots` not stores [index, bool] so that we can
track what's being used by the channel.

`get_candidate_slots`:
- Returns all the slots that a channel may use, even if they
  are occupied 
- When we're first assigning slots, they're of course all 
  unoccupied

`get_usable_slots`:
- Get the number of slots we need
- Get all the slots that the channel can use
- Filter out any slots that are occupied in the global
  map
  Q: if we return the full [u16, bool] we could assert
     that they're always in the same state?
- No slots if there aren't enough
- Otherwise just take the number we need

`may_add_htlc`:
- Unchanged, just checks that we get them

`add_htlc`:
- Gets the slots that we can use
- Assign each of the slots to being used:
  - Set index in the global map
  - Find index in the channel's map
  - Assert that they're consistent

`remove_htlc`:
- Get all slots that the channel can use
- Filter out to only get the ones that are currently occupied
- Check that enough are occupied
- For each of the occupied slots on the channel map:
  - Set the global map to not used
    Q: we should assert here that it was true 
  - Check that we can find the slot on the channel and
    set it to unoccupied
