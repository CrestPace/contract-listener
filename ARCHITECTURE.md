# Contract Listener Technical Documentation

## 1. Overview

The **contract-listener** is a background service responsible for bridging on-chain activity with off-chain application infrastructure. It continuously monitors our smart contracts for specific event logs and error conditions, fetches live token price metrics, and reliably pushes structured data into the main backend system. The service is built for high concurrency and resilience, leveraging Rust’s async ecosystem.

## 2. Key Technologies

| Component        | Technology / Crate | Purpose |
|------------------|-------------------|---------|
| Runtime          | **Tokio**         | Async event loop, task scheduling, I/O multiplexing |
| Blockchain RPC   | **Alloy**         | Type-safe contract interaction, log subscription, transaction receipt queries |
| Caching          | **Redis**         | Fast in-memory store for token prices and temporary state |
| Serialization    | **serde / serde_json** | Data marshalling for backend communication |
| Backend client   | **reqwest** (or gRPC) | HTTP/gRPC client to deposit processed data |

## 3. System Architecture

```text
┌────────────────────────┐
│     RPC Provider       │
│  (Ethereum / L2 node)  │
└──────────┬─────────────┘
           │  Alloy (WebSocket / HTTP)
           ▼
┌──────────────────────────────────────────┐
│             Contract Listener            │
│                                          │
│  ┌────────────────────────────────────┐  │
│  │   Event & Error Log Subscriber     │  │
│  │  (Alloy Provider + Contract bind.) │  │
│  └───────────────┬────────────────────┘  │
│                  │                       │
│  ┌───────────────▼────────────────────┐  │
│  │        Processing Pipeline         │  │
│  │  • Decode events                   │  │
│  │  • Enrich with price data (Redis)  │  │
│  │  • Transform to backend format     │  │
│  └───────────────┬────────────────────┘  │
│                  │                       │
│  ┌───────────────▼────────┐              │
│  │      Redis Cache       │              │
│  │  (Live token prices)   │              │
│  └────────────────────────┘              │
│                  │                       │
│  ┌───────────────▼────────────────────┐  │
│  │      Backend Data Depositor        │  │
│  │  (HTTP/gRPC calls to main backend) │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
           │
           ▼
┌────────────────────────┐
│    Main Application    │
│        Backend         │
└────────────────────────┘
```

## 4. Core Components

### 4.1 Blockchain Log Subscriber

- **Implementation**: Uses Alloy’s `Provider` with a WebSocket transport to subscribe to contract event logs (`eth_subscribe` for `logs`).
- **Event listening**: A set of pre-configured contract addresses and event signatures (topics) are monitored. Incoming logs are decoded using Alloy’s Solidity type bindings.
- **Error log monitoring**:  
  - Some contracts emit custom `Error` events; these are caught via the same log subscription.  
  - For generic transaction errors (e.g., reverts), the listener can poll transaction receipts for failed status, or subscribe to `eth_subscribe("newHeads")` and inspect block transactions. The exact strategy is configurable.  
  - All error-related data is enriched with block context and forwarded to the backend.

### 4.2 Redis Cache Manager

- **Purpose**: Maintain a fresh in-memory cache of token price metrics. This avoids repeated calls to external price APIs and provides low-latency enrichment for every processed event.
- **Data model**:  
  - Key pattern: `price:{token_address}:{currency}` (e.g., `price:0x...:USD`)  
  - Value: JSON containing `price`, `timestamp`, `source`.  
- **Populating**: A periodic async task (every 10 seconds, configurable) fetches prices from trusted oracles/exchanges and updates Redis.  
- **Consumption**: The processing pipeline reads current prices using Redis `GET` commands or an in-memory mirror updated via pub/sub or periodic sync.

### 4.3 Processing Pipeline

This is the heart of the listener. It performs the following for each received log:

1. **Decode** the raw log into a structured event (e.g., `Transfer`, `Swap`).
2. **Lookup** relevant token prices from Redis.
3. **Transform** the event + price data into a payload matching the main backend’s API contract.
4. **Buffer & batch** (if configured) for efficient backend writes.
5. **Forward** to the Backend Data Depositor.

### 4.4 Backend Data Depositor

- An async component that sends processed data to the main application backend.
- It can be an HTTP JSON client, a gRPC client, or a message queue producer. The documentation assumes a REST/JSON interface.
- Implements retries with exponential backoff, circuit breaker logic, and dead-letter queuing for undeliverable payloads.
- Designed to handle backpressure through Tokio’s bounded channels and batch accumulation.

## 5. Data Flow Walkthrough

1. The listener maintains a persistent WebSocket connection to the RPC provider via Alloy.
2. When a new block contains a matching event log, Alloy’s subscription stream yields a `Log` object.
3. The log is sent through an `mpsc` channel to a processing worker pool.
4. A worker decodes the log, fetches the relevant price(s) from Redis (with a timeout), and assembles the backend payload.
5. The payload is placed onto a sender channel connected to the depositor.
6. The depositor sends the payload to the backend API. On success, it acknowledges processing; on failure, it re-queues with backoff.
7. The price fetcher task runs independently, periodically updating Redis keys.

## 6. Caching Strategy for Live Price Metrics

- **Cache-aside**: The service reads from Redis; if a key is missing, it may trigger an immediate fetch from the price source (optionally with a short lock to avoid thundering herd).
- **TTL**: Each price entry carries a TTL (e.g., 10 seconds) to ensure stale data is not used. Redis `SETEX` or `SET` with `EX` is used.
- **Consistency**: Because prices are refreshed on a fixed interval, the listener may serve slightly delayed data. This is acceptable for the use case.

## 7. Error Handling and Resilience

- **RPC disconnections**: Alloy’s WebSocket transport handles automatic reconnection with backoff. The listener logs reconnection attempts and replays missed blocks if necessary (using a block height cursor stored in Redis).
- **Redis failures**: The service degrades gracefully – it can queue events with a flag indicating missing price enrichment, or use a cached in-process last-known price for a short period.
- **Backend outages**: The depositor uses persistent retries and an optional disk-backed dead-letter log to avoid data loss.
- **Panic safety**: Tokio tasks are spawned with `tokio::spawn`, and individual task panics are caught by a supervising task that logs and restarts them.

## 8. Configuration

The contract-listener is configured via environment variables or a YAML file:

| Parameter                 | Description |
|---------------------------|-------------|
| `RPC_WS_URL`              | WebSocket endpoint of the blockchain node |
| `CONTRACTS`               | Comma-separated list of `address:abi_path` pairs |
| `EVENT_SIGNATURES`        | List of event topics to monitor |
| `ERROR_MONITORING`        | Strategy (`event`, `receipt`, `head`) |
| `REDIS_URL`               | Redis connection string |
| `PRICE_UPDATE_INTERVAL_S` | Seconds between price fetches |
| `BACKEND_URL`             | Main backend API endpoint |
| `BACKEND_BATCH_SIZE`      | Maximum events to batch in one request |
| `BACKEND_RETRY_LIMIT`     | Max retry attempts per payload |

## 9. Running the Service

The service is built as a single Rust binary. Typical execution:

```bash
cargo run --release
```

It starts Tokio, initializes the Redis connection pool, establishes the RPC subscription, spawns the price fetch loop and depositor, and awaits shutdown signals (SIGINT/SIGTERM) to gracefully drain in-flight work.

## 10. Monitoring and Logging

Structured logs via the tracing crate; log levels configurable.
Metrics exposed via an optional Prometheus endpoint (e.g., events processed per second, cache hit ratio, backend latency, queue depths).
Health check endpoint to verify Redis connectivity and RPC subscription status.
## 11. Extending the Listener

To add a new contract:

Place the contract ABI in the abis/ directory.
Add its address and event signatures to the configuration.
The decoder uses the ABI to parse logs automatically.
If custom enrichment logic is needed, implement a new Processor trait and register it in the pipeline.
## 12. Dependencies (Cargo.toml)

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
alloy = { version = "2.1", features = ["full"] }
redis = { version = "0.25", features = ["tokio-comp"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
reqwest = { version = "0.12", features = ["json"] }
tracing = "0.1"
tracing-subscriber = "0.3"
futures = 0.3
```
