# Fully Deterministic Produce Events

Issue: https://github.com/bitcoin-dev-project/sim-ln/issues/243
(tl;dr: threads get locks in different orders, making undeterministic

New struct: Payment Event
- Contains the details of the next payment event, and an executor
  so that we can continue to make events if necessary
- This implements ordering so that we can track them in a queue

Q: Why does it manually implement Eq?
- Because we then have to go and derive Eq for all of the things that
  we hold in the ExecutorKit (which includes des gen, payment gen etc)

Q: Why does DestinationGenerator need to be Sync now?

Q: Why do we need to use a BTreeMap for our list of nodes now?
