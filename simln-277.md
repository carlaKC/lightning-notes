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
