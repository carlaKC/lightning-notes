# Debugging Forward Builder

Problem: We're getting _way less_ than I expect when we run 
forward-builder.

Approach outlined in: https://github.com/carlaKC/jam-ln/issues/90

## Checking Two Node Network

Running with the following:
```
{
  "sim_network": [
    {
      "scid": 123,
      "capacity_msat": 10000000000,
      "node_1": {
        "pubkey": "035a43121d24b2ff465e85af9c07963701f259b5ce4ee636e3aeb503cc64142c11",
        "alias": "0",
        "max_htlc_count": 483,
        "max_in_flight_msat": 10000000000,
        "min_htlc_size_msat": 1,
        "max_htlc_size_msat": 9999000000,
        "cltv_expiry_delta": 40,
        "base_fee": 0,
        "fee_rate_prop": 0
      },
      "node_2": {
        "pubkey": "0242902a3a5aa34829db9def5b44939f9f459f4ee08e97cba18516c62ddf8ec9e6",
        "alias": "1",
        "max_htlc_count": 15,
        "max_in_flight_msat": 10000000000,
        "min_htlc_size_msat": 1,
        "max_htlc_size_msat": 9999000000,
        "cltv_expiry_delta": 144,
        "base_fee": 1000,
        "fee_rate_prop": 1000
      }
    },
    {
      "scid": 456,
      "capacity_msat": 10000000000,
      "node_1": {
        "pubkey": "0242902a3a5aa34829db9def5b44939f9f459f4ee08e97cba18516c62ddf8ec9e6",
        "alias": "1",
        "max_htlc_count": 15,
        "max_in_flight_msat": 10000000000,
        "min_htlc_size_msat": 1,
        "max_htlc_size_msat": 9999000000,
        "cltv_expiry_delta": 144,
        "base_fee": 1000,
        "fee_rate_prop": 1000
      },
      "node_2": {
        "pubkey": "02f3b7c18e4e94c7df1f071cdf1f49cbde7cc508e4b7dbb5a1f0d5e4644b23e3f1",
        "alias": "2",
        "max_htlc_count": 483,
        "max_in_flight_msat": 10000000000,
        "min_htlc_size_msat": 1,
        "max_htlc_size_msat": 9999000000,
        "cltv_expiry_delta": 40,
        "base_fee": 0,
        "fee_rate_prop": 0
      }
    }
  ]
}
```

Set the target.txt = 1, so that the target node is ignored and the
simulator will send payments back and forth between the nodes.

Generated peacetime data for 31 days:
`forward-builder --network-dir data/two_node --traffic-type peacetime --duration 31d `

GPT-ed a script that'll add up the total on each outgoing channel.
Since we know these are *always* two hop routes, the outgoing amount
is the total amount sent.

```
#!/usr/bin/env python3
"""
Script to analyze Lightning Network forward data by channel_out_id.
"""

import csv
import sys
from collections import defaultdict

def analyze_forwards_by_channel(file_path: str):
    """Analyze forward data by channel_out_id."""
    
    # Data storage: channel_out_id -> {'total_amount': int, 'count': int}
    channel_stats = defaultdict(lambda: {'total_amount': 0, 'count': 0})
    
    # Read CSV data
    try:
        with open(file_path, 'r') as f:
            reader = csv.DictReader(f)
            for row in reader:
                outgoing_amt = int(row['outgoing_amt'])
                channel_out = row['channel_out_id']
                
                channel_stats[channel_out]['total_amount'] += outgoing_amt
                channel_stats[channel_out]['count'] += 1
                
    except FileNotFoundError:
        print(f"Error: File '{file_path}' not found")
        return
    except Exception as e:
        print(f"Error reading file: {e}")
        return
    
    if not channel_stats:
        print("No data found in file")
        return
    
    print("Channel Out ID Analysis")
    print("=" * 60)
    print(f"{'Channel ID':<15} {'Total Sent (msat)':<20} {'Count':<10} {'Avg per Forward':<15}")
    print("-" * 60)
    
    # Sort by total amount descending
    for channel_id, stats in sorted(channel_stats.items(), 
                                   key=lambda x: x[1]['total_amount'], 
                                   reverse=True):
        total = stats['total_amount']
        count = stats['count']
        avg = total / count if count > 0 else 0
        
        print(f"{channel_id:<15} {total:>19,} {count:>9} {avg:>14,.0f}")
    
    # Summary
    total_forwards = sum(stats['count'] for stats in channel_stats.values())
    total_amount = sum(stats['total_amount'] for stats in channel_stats.values())
    
    print("-" * 60)
    print(f"{'TOTAL':<15} {total_amount:>19,} {total_forwards:>9} {total_amount/total_forwards:>14,.0f}")
    print(f"\nUnique channels: {len(channel_stats)}")

if __name__ == "__main__":
    if len(sys.argv) != 2:
        print("Usage: python3 analyze_forwards.py <csv_file>")
        sys.exit(1)
    
    analyze_forwards_by_channel(sys.argv[1])
```

Results from running for 1 month:
```
Channel Out ID Analysis
============================================================
Channel ID      Total Sent (msat)    Count      Avg per Forward
------------------------------------------------------------
123                   4,549,997,777      1337      3,403,140
456                   1,540,596,941      1374      1,121,250
------------------------------------------------------------
TOTAL                 6,090,594,718      2711      2,246,623

Unique channels: 2
```

This looks around correct for channel 123:
```
[simln_lib] Starting activity producer for 0(035a43...142c11): activity generator for capacity: 5000000000 with multiplier 1: 1315.7894736842106 payments per month (1.8274853801169593 per hour).
[simln_lib] Starting activity producer for 2(02f3b7...23e3f1): activity generator for capacity: 5000000000 with multiplier 1: 1315.7894736842106 payments per month (1.8274853801169593 per hour).
2
```

We expect around 5,000,000,000 _per_ node (per logs above).
Our network is:
`Node0 -(123)-> Node1 -(456)-> Node2`

The analysis looks at outgoing node, so we know:
- The correct amount is going from 2 -> 1 -> 0 (90% of target)
- Too little is going from 0 -> 1 -> 2 (30% of target)

We also end with an success rate of 99.67% in this simple network, so 
we're not running into failures. The nodes have the same capacity, and
they're each other's destinations so they should have exactly the same
parameters in our random amount generation.

The simulator processed 2711 payments in a month, sim-ln says that it
should be processing 1315 per node so this is on track with the right
amount.

There are 2712 lines in the peacetime file (2711 + header), so we're
definitely writing all of the forwards that succeed to disk. 

Checked parameters for this:
- mu: 7.9683195000127185 
- sigma_square: 14.364384249367792 
- std_dev: 4999998555

[blocked] Running this a second time

