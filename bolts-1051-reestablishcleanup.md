# Rework Channel Reestablish Requirements

Start after we've exchanged `channel_ready`:
- A last commitment_signed sent: 1
- A last commitment_signed received: 1
- A last revoked_and_ack: 0

- B last commitment_signed sent: 1
- B last commitment_signed received: 1
- B last revoked_and_ack: 0

#### Commitment Signed not Received
Now we exchange some messages:
```
A --- update_add_htlc ---> B
A -- commitment_signed (x) B
```

The connection cycles before B receives commitment_signed.

- A last commitment_signed sent: 2
- A last commitment_signed received: 1
- A last revoke_and_ack: 0

- B last commitment_signed sent: 1
- B last commitment_signed received: 1
- B last revoke_and_ack: 0

##### Channel Reestablish Messages:
A sends to B:
- next_commitment_number: 2
- next_revocation_number: 1

B sends to A:
- next_commitment_number: 2
- next_revocation_number: 1

##### Channel Reestablish Checks:
A checks:
- next_revocation_number > last commitment_signed: 
  false: 1 > 2
- last commitment_signed > next_revocation_number +1: 
  false: 2 > 1 + 1
- next_commitment_number > last commitment_signed + 1:
  false: 2 > 2 + 1
- next_commitment_number < commitment_signed:
  false: 2 < 2
- next_commitment_number == commitment_signed: 
  true: 2 == 2
- next_revocation_number == last revoke_and_ack:
  false: 1 == 0

B checks:
- next_revocation_number > last commitment_signed: 
  false: 1 > 
- last commitment_signed > next_revocation_number +1:
  false: 1 > 1 + 1
- next_commitment_number > last commitment_signed + 1:
  false: 2 > 1
- next_commitment_number < commitment_signed:
  false: 2 < 1
- next_commitment_number == commitment_signed:
  false: 2 == 1
- next_revocation_number == last revoke_and_ack:
  false: 1 == 0

-> A will send commitment_signed with number 2 because B did not 
   receive it.
#### Commitment Signed Received
A now successfully manages to sent commitment_signed to B, re-using 
commitment number 2.

```
A -- commitment_signed --> B
```

B received the commitment_signed from A, then the connection cycles:
- A last commitment_signed sent: 2
- A last commitment_signed received: 1
- A last revoke_and_ack: 0

- B last commitment_signed: 1
- B last commitment_signed received: 2
- B last revoke_and_ack: 0

The connection cycles, and they have to send channel re-establish:

##### Channel Reestablish Messages:
A sends to B:
- next_commitment_number: 2
- next_revocation_number: 1

B sends to A:
- next_commitment_number: 3
- next_revocation_number: 1

##### Channel Reestablish Checks:
A checks the following:
- next_revocation_number > last commitment_signed: 
  false: 1 > 2
- last commitment_signed > next_revocation_number +1: 
  false: 2 > 1 + 1
- next_commitment_number > last commitment_signed + 1:
  false: 3 > 2 + 1
- next_commitment_number < commitment_signed:
  false: 3 < 2
- next_commitment_number == commitment_signed: 
  false: 3 == 2
- next_revocation_number == last revoke_and_ack:
  false: 1 == 0

B checks the same:
- next_revocation_number > last commitment_signed: 
  false: 1 > 
- last commitment_signed > next_revocation_number +1:
  false: 1 > 1 + 1
- next_commitment_number > last commitment_signed + 1:
  false: 2 > 1
- next_commitment_number < commitment_signed:
  false: 2 < 1
- next_commitment_number == commitment_signed:
  false: 2 == 1
- next_revocation_number == last revoke_and_ack:
  false: 1 == 0

-> There's no action for either node, they're on the same page.

#### Revoke not received
Now, B finalized their commitment update by revoking:
```
A (x) revoke_and_ack <-- B
```

A does not receive the message before the connection cycles:
- A last commitment_signed sent: 2
- A last commitment_signed received: 1
- A last revoked_and_ack: 0

- B last commitment_signed sent: 1
- B last commitment_signed received: 1
- B last revoked_and_ack: 1

##### Channel Reestablish Messages:
A sends to B:
- next_commitment_number: 2
- next_revocation_number: 1

B sends to A:
- next_commitment_number: 3
- next_revocation_number: 1

##### Channel Reestablish Checks:
A checks the following:
- next_revocation_number > last commitment_signed: 
  false: 1 > 2
- last commitment_signed > next_revocation_number +1: 
  false: 2 > 1 + 1
- next_commitment_number > last commitment_signed + 1:
  false: 3 > 2 + 1
- next_commitment_number < commitment_signed:
  false: 3 < 2
- next_commitment_number == commitment_signed: 
  false: 3 == 2
- next_revocation_number == last revoke_and_ack:
  false: 1 == 0

B checks the same:
- next_revocation_number > last commitment_signed: 
  false: 1 > 1
- last commitment_signed > next_revocation_number +1:
  false: 1 > 1 + 1
- next_commitment_number > last commitment_signed + 1:
  false: 2 > 1 + 1
- next_commitment_number < commitment_signed:
  false: 2 < 1
- next_commitment_number == commitment_signed:
  false: 2 == 1
- next_revocation_number == last revoke_and_ack:
  true: 1 == 1

-> B must retransmit their last revoke_and_ack

#### Revoke Received
Now, B retransmits the revoke_and_ack and A receives it.
```
A <-- revoke_and_ack -- B
```

A receives the message before the connection is cycled
- A last commitment_signed sent: 2
- A last commitment received: 1
- A last revoked_and_ack: 0

- B last commitment_signed sent: 1
- B last commitment_signed received: 2
- B last revoked_and_ack sent: 1

##### Channel Reestablish Messages:
A sends to B:
- next_commitment_number: 2
- next_revocation_number: 2

B sends to A:
- next_commitment_number: 3
- next_revocation_number: 1

##### Channel Reestablish Checks
A checks the following:
- next_revocation_number > last commitment_signed: 
  false:  1 > 2
- last commitment_signed > next_revocation_number +1: 
  false: 2 > 1 + 1
- next_commitment_number > last commitment_signed + 1:
  false: 3 > 2 + 1
- next_commitment_number < commitment_signed:
  false: 3 < 2
- next_commitment_number == commitment_signed: 
  false: 3 == 2
- next_revocation_number == last revoke_and_ack:
  false: 1 == 0 

B checks the same:
- next_revocation_number > last commitment_signed: 
  false: 2 > 1 
- last commitment_signed > next_revocation_number +1:
  false: 1 > 2 + 2 
- next_commitment_number > last commitment_signed + 1:
  false: 2 > 1 + 1 
- next_commitment_number < commitment_signed:
  false: 2 < 1 
- next_commitment_number == commitment_signed:
  false: 2 == 1 
- next_revocation_number == last revoke_and_ack:
  false: 2 == 1

-> No action from either node, they're on the same page.
