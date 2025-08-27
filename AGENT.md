# MEV Agent Documentation

## 1. Overview

This document outlines the architecture and operational flow of the Jito MEV (Maximal Extractable Value) Agent. This system is a high-frequency arbitrage bot designed to operate on the Solana blockchain. It identifies and executes profitable opportunities, primarily through flash-loan-based arbitrage, by submitting atomic transaction bundles to the Jito Labs block engine.

This is not a toy. It is a production-grade, low-latency system engineered for a hyper-competitive environment. Every architectural decision has been made to minimize latency, maximize security, and ensure operational robustness.

---

## 2. Architectural Philosophy

The agent is built upon a set of core principles that guide its design and future development.

*   **Latency is the Enemy**: In the MEV space, speed is synonymous with profitability. Our architecture reflects a relentless pursuit of latency reduction through infrastructure co-location, a dedicated RPC node, and an event-driven design.

*   **React, Don't Ask**: Polling for state changes is an anti-pattern that guarantees you are already too late. The agent is designed as a reactive system. It subscribes to a direct, real-time stream of on-chain events and acts instantly upon them.

*   **Security is Non-Negotiable**: A profitable system is a target. We operate on a zero-trust basis for sensitive materials. All secrets are managed by a hardened, external service (GCP Secret Manager) and are never stored in source control or on-disk in production.

*   **Modularity and Extensibility**: The system is composed of logical, decoupled components. The core Rust engine is responsible for speed and execution, while the strategy can be augmented by external signal generators (e.g., Python ML models) via a clean gRPC interface. This allows for independent development and iteration of components.

*   **Simulate Before You Act**: Never risk capital on an unverified hypothesis. Every potential opportunity is subjected to a rigorous simulation and risk-gating process before being committed to the network. We verify profitability and execution safety before sending.

---

## 3. System Components

The system is a vertically integrated stack, from the infrastructure to the application logic.

### 3.1. Rust Core (`jito-mevbot`)

The heart of the operation. A highly-performant, asynchronous Rust application responsible for execution.

*   **`main.rs`**: The orchestrator. It initializes all components, establishes the connection to the Geyser event stream, and manages the main event-driven loop.
*   **`strategy.rs`**: The "brain". It contains the logic for identifying arbitrage opportunities from cached market data and constructing the atomic transaction bundles.
*   **`state.rs`**: Defines the `AccountCache`, a thread-safe, in-memory cache that holds the real-time state of all monitored on-chain accounts.
*   **`protocols.rs`**: The on-chain "address book". It provides canonical public keys and `borsh`-compatible data structures for all protocols the agent interacts with. This centralization is critical for maintainability.
*   **`auth.rs`**: A secure, environment-aware module for loading the bot's private keys. It intelligently switches between a local `.env` file (for development) and GCP Secret Manager (for production).

### 3.2. Dedicated RPC Node (`jito-solana`)

A standard Solana validator node, built from the `jito-solana` source, running in **non-voting RPC mode**. This node is co-located in the same data center as the bot. Its sole purpose is to provide the bot with a private, ultra-low-latency window into the Solana network. It runs the **Geyser gRPC Plugin**, which produces the real-time event stream our agent consumes.

### 3.3. External Signal Generator (Optional)

The agent is architected to receive signals from an external source via gRPC (see `proto/signals.proto`). This allows for complex, computationally-intensive models (e.g., in Python) to analyze market data and provide trading signals to the Rust core without compromising the core's performance.

---

## 4. Operational Flow

The agent's lifecycle for a single arbitrage event is as follows:

1.  **Event**: A transaction modifies a monitored liquidity pool (e.g., Raydium SOL-USDC) on the Solana network.
2.  **Propagation**: The co-located `jito-solana` node processes this block.
3.  **Push**: The Geyser gRPC plugin on the node immediately serializes the account update and pushes it over a persistent connection to the Rust agent.
4.  **Ingestion**: The Geyser listener task in `main.rs` receives the update.
5.  **Cache & Signal**: The listener task writes the new account data to the shared `AccountCache` and sends a lightweight signal via an `mpsc` channel to the main loop, waking it up.
6.  **Analysis**: The main loop calls `strategy.find_and_build_bundle()`, which reads the fresh data from the `AccountCache`.
7.  **Construction**: If a profitable arbitrage is identified, the strategy module constructs a complete, atomic transaction bundle. This includes the flash loan borrow, the two swaps, the loan repayment, and the Jito tip.
8.  **Simulation**: The bundle is passed to `strategy.simulate_and_gate_bundle()`. A `simulateBundle` RPC call is made to the Jito block engine.
9.  **Gating**: The simulation response is checked for execution errors, profitability (by parsing balance changes), and excessive Compute Unit usage.
10. **Execution**: If the bundle passes all gates, it is sent to the Jito block engine via `sendBundle` to be included in the next MEV auction.

This entire sequence, from on-chain event to bundle submission, is optimized to occur in a few milliseconds.

---

## 5. Setup & Deployment

### 5.1. Infrastructure

The cloud infrastructure is defined in `main.tf`. It should be provisioned using Terraform. This will create:
*   A compute-optimized GCP instance for the bot.
*   Firewall rules (which should be locked down to specific IPs).
*   The necessary GCP Secret Manager entries.

A separate, high-performance machine must be provisioned to run the `jito-solana` RPC node.

### 5.2. Secrets

Before first run, populate the secrets in GCP Secret Manager:
*   `SOLANA_PRIVATE_KEY`: The bot's hot wallet private key in base58 format.
*   `HELIUS_API_KEY`: Your Helius (or other RPC provider) API key for fallback operations.

For local development, create a `.env` file from `.env.example` and populate it. **DO NOT COMMIT `.env` TO SOURCE CONTROL.**

### 5.3. Running the Agent

1.  Build the release binary: `cargo build --release`.
2.  Install the `systemd` service file located in the repository to `/etc/systemd/system/mevbot.service`.
3.  **Core Affinity**: The `systemd` file should use `taskset` to bind the agent process to specific, dedicated CPU cores. This is critical for minimizing latency variance. Example: `ExecStart=taskset -c 4,5,6,7 /path/to/jito-mevbot/target/release/jito-mevbot`.
4.  Enable and start the service:
    ```bash
    sudo systemctl enable mevbot.service
    sudo systemctl start mevbot.service
    ```
5.  Monitor logs: `journalctl -u mevbot -f`.

---

## 6. Development & Contribution

The primary development task is the implementation of the transaction-building logic within `strategy.rs`. The current code contains `PLACEHOLDER` comments where protocol-specific instruction builders are required.

*   **Instruction Accuracy**: Use the official SDKs or source code of target protocols (Solend, Raydium, Orca, etc.) to build byte-perfect transaction instructions. Any error here will cause the entire bundle to fail.
*   **Simulation is Mandatory**: All new strategies or modifications MUST be tested against a forked mainnet environment before being considered for production. Use `solana-test-validator --url <RPC_URL> --fork <SLOT>` to create a realistic testbed.
*   **Tuning**: The values for priority fees, tip percentages, and profit gates are not static. They are critical variables that must be tuned based on performance analysis and network conditions.
