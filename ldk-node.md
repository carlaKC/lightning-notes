# LDK node

Build node once configured to pref (`build`):
- Creates a new sqlite store 
- `build_with_store`:
  - setup logger and seed bytes
  - `build_with_store_internal`:
    - Private key creation 
    - Setup a BDK wallet with `KVStoreWalletPersister`
    - Setup Esplora chain client and tx services 
    - Create `ChainMonitor` with tx services (broadcast, fee est) and db
    - Create `KeysManager` with seed
    - Read network graph, scorer and router from db (if any)
    - Read `ChannelMonitor` from store (if any)
    - Create `ChannelManager` from store or fresh
    - Report each `ChannelMonitor` to `ChainWatcher`
    - Setup peer manager:
      - `ChannelManager` handles channel related messages
      - Setup appropriate gossip handler for routing 
      - Add onion and custom message handling
    - Setups stores for payments, events and peers

What's getting persisted (ie, things that we pass `kv_store` to?
- Wallet
- Network Graph 
- Pathfinding scores
- Channel monitors
- Channel manager
- Gossip
- Output sweeper
- Payment store
- Event queue
- Peer store

Q: what's the difference between channel monitors and channel manager?
- ChannelMonitor is reported to ChainWatcher

`start_with_runtime`:
- Startup tasks
  - Once-off: initiate current fee rate
  - Ticker: update fee rate estimate
  - Ticker: sync on-chain wallet
  - If RGS: pull latest gossip from server
  - Loop: listen for incoming TCP connections, connect to `PeerManager`
  - Ticker: periodically connect to persisted peers 
  - Ticker: broadcast channel announcements
  - Ticker: broadcast on-chain transactions
- Listen for shutdown triggers 
- Run LN background processor with event handler
- Spin up a liquidity handler (LSP) if one is present

