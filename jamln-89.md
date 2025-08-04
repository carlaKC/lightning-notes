# Run without bootstrap

Where do we use bootstrap?
- `reputation_summary`: get the file we're dealing with
  - Trivial, the function wants the bootstrap anyway
- Validate CLI: trivial
- `RevenueInterceptor::new_with_bootstrap`:
  - Passed through to `new_with_bootstrap` of `PeacetimeRevenue`
