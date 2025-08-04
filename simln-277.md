# Deterministic Events

Adding a heap of payment events to improve determinism:
- `generate_payment`: adds that payment to the queue
  - Once on start up, once for every removed event (if required)
  - We create a `PaymentEventPaylaod` for each executor
  - We push it to a hashmap of executors etc
    Q: why does this have destination etc in it?
    Q: why is the destination the source?
  - We then call `generate_payment` with the heap,
    the individual executor and the payloads?

What do we need from the hashmap?
- We just lookup the starting payload, and then overwrite
  the values in `generate_payment`

## Review Iteration 2

- `DestinationGenerator` is `Sync` now?
  - Arc is only `Send` if its contents are `Sync`, because moving
    an arc to another thread means that there's possibility that 
    multiple threads are using it
    (we maybe wouldn't have hit this if using `Box` for getting dyn
    traits to be sized).

`PaymentEvent`:
- Contains the details about the next event

`PaymentEventPayload`:
- Contains the meta information needed to dispatch the payment,
  bad name

`random_activity_nodes`:
- Sort the nodes in the network to crate a `network_generator`
  - This is necessary so that we always fill the same public key
    in the same place for our weighted index.

`dispatch_producers`:
- Takes the executors and a sender channel for outputs
