# The architectural guide for the contract-listener:

The contract-listener is built with Rust with a standard Tokio crate for async handling, Redis for caching, and Alloy for communicating with the RPC provider to listen to event logs and error logs from the blockchain.

The contract-listener will be able to do a couple of things, including:
1. Hold the cache from fetching live price metrics for tokens.
2. Hold and deposit data to the major backend of the application after listening to the event logs from the RPC provider.
