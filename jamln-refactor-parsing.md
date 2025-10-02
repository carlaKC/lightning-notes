# Refactoring Parsing

Our parsing is very messy/confusing at the moment.

Here's what I think our topology should look like:
```
networks/
  {network_name}/
    peacetime_network.json
    peacetime_traffic.csv
    reputation.csv
    target.txt
    attacks/
      {attack_name}/
        attacktime_network.json
        attacker.csv
        attacktime_traffic.csv
        reputation/
          reputation_{duration}.csv 
          revenue_{duration}.csv 
```

I also think we should separately have a `results` directory:
```
results/
  {network_name}/
    {attack}/
      {timestamp}/
         summary.txt
```

Now, what do each of our "modes" need for this:

main/attack:
- peacetime_network.json (to validate graph)
- peacetime_traffic.csv (to replay peacetime)
- reputation.csv (to bootstrap network)
- target.txt (to identify target for attack)
- attacktime_network.json (to run graph)
- attacker.csv (to run attack)
   
main/attack - bootstrap:
- All of the above (^)
- reputation_{duration}.csv 
- revenue_{duration}.csv 

forward_builder - peacetime:
- peacetime_network.json
- target.txt
-> outputs: {network_name}/peacetime_traffic.csv

forward builder - attacktime:
- peacetime_network.json (to validate graph)
- target.txt (to identify target for attack)
-> outputs: {network_name}/attacks/{attack_name}/attacktime_traffic.csv 

reputation builder - no bootstrap:
- peacetime_network.json (to build reputation) 
-> outputs: {network_name}/reputation.csv

reputation builder - with bootstrap:
- attacktime_traffic.json (to build reputation)
- attacker.csv
-> outputs: {network_name}/attacks/{attack_name}/reputation_{duration}.csv
-> outputs: {network_name}/attacks/{attack_name}/revenue_{duration}.txt

What are my key components?
- Peacetime network:
  - peacetime_network.json
  - peacetime_traffic.csv
  - reputation.csv
  - target.txt

- Attacktime network:
  - attacktime_traffic.sjon
  - attacker.csv

- Bootstrap info:
  - reputation_{duration}.csv
  - revenue_{duration}.txt

What do each of them consist of?
main/attack: peacetime + attacktime
main/attack/bootstrap: peacetime + attacktime + bootstrap
forward_builder/peacetime: peacetime
forward_builder/attacktime: peacetime + attacktime 
reputation builder/no bootstrap: peacetime
reputation builder/bootstrap: peacetime + attacktime + bootstrap

Checks for these changes:
[x] Create reputation for peacetime network from scratch
    [x] Traffic
    [x] Reputation
[x] Create reputation for bootstrap network from scratch
    [x] Traffic
    [x] Reputation
[x] Run main with no attack set
[x] Run forward builder with bootstrap set
[x] Run reputation builder with attack no bootstrap
[?] Run sink attack with no bootstrap
    -> Target has 11/39 pairs reputation; did this change?
[x] Run sink attack with repuation (different values)
   [x] 30d just checked that it runs
   [x] 90d running now, seems to be working!
[x] Check writing of results
