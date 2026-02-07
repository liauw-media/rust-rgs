# rust-rgs

Remote Gaming Server (RGS) implementation in Rust for GLI-19 compliant slot games.

## Overview

This is the core RGS server that orchestrates game sessions, manages state, handles wallet integration, and coordinates with game engines. It replaces the TypeScript `@omnitronix/rgs` package.

**Reference Implementation:** [`omnitronix-rgs/`](https://github.com/liauw-media/omnitronix-monorepo) (TypeScript/NestJS)

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              rust-rgs                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  API Layer (Axum):                                                      │
│  ├── /api/v1/games/:gameCode/session       (POST - start session)     │
│  ├── /api/v1/games/:gameCode/spin          (POST - execute spin)      │
│  ├── /api/v1/games/:gameCode/bonus         (POST - bonus actions)     │
│  └── /api/v1/health                        (GET - health check)       │
│                                                                         │
│  Session Management:                                                    │
│  ├── SessionStore (Redis-backed)                                        │
│  ├── State serialization (public/private separation)                   │
│  └── Session timeout handling                                           │
│                                                                         │
│  Game Engine Registry:                                                  │
│  ├── Dynamic engine loading (WASM or gRPC)                             │
│  ├── Engine version management                                          │
│  └── Hot reload support                                                 │
│                                                                         │
│  Wallet Integration:                                                    │
│  ├── Operator wallet API client                                         │
│  ├── Transaction management                                             │
│  └── Rollback handling                                                  │
│                                                                         │
│  Audit & Compliance:                                                    │
│  ├── GLI-19 audit trail                                                 │
│  ├── RNG outcome hashing                                                │
│  └── Game round recording                                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
          │                    │                    │
          ▼                    ▼                    ▼
   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
   │ rust-rng    │     │ Game Engine │     │  Operator   │
   │ (gRPC)      │     │ (WASM/gRPC) │     │  Wallet     │
   └─────────────┘     └─────────────┘     └─────────────┘
```

## Core Components

### Session Management

```rust
/// Game session state
pub struct GameSession {
    pub session_id: Uuid,
    pub player_id: String,
    pub game_code: String,
    pub currency: Currency,
    pub created_at: DateTime<Utc>,
    pub last_activity: DateTime<Utc>,
    pub public_state: serde_json::Value,
    pub private_state: serde_json::Value,  // Encrypted at rest
    pub active_bonus: Option<BonusSession>,
}

/// Session store trait
#[async_trait]
pub trait SessionStore: Send + Sync {
    async fn create(&self, session: GameSession) -> Result<(), SessionError>;
    async fn get(&self, session_id: Uuid) -> Result<Option<GameSession>, SessionError>;
    async fn update(&self, session: GameSession) -> Result<(), SessionError>;
    async fn delete(&self, session_id: Uuid) -> Result<(), SessionError>;
    async fn cleanup_expired(&self) -> Result<u64, SessionError>;
}

/// Redis-backed session store
pub struct RedisSessionStore {
    pool: Pool<RedisConnectionManager>,
    ttl: Duration,
    encryption_key: [u8; 32],
}
```

### Game Engine Registry

```rust
/// Registry for game engines
pub struct GameEngineRegistry {
    engines: HashMap<String, Arc<dyn GameEngineAdapter>>,
}

/// Adapter trait for different engine loading strategies
#[async_trait]
pub trait GameEngineAdapter: Send + Sync {
    async fn process_command(
        &self,
        public_state: Option<serde_json::Value>,
        private_state: Option<serde_json::Value>,
        command: GameActionCommand,
    ) -> Result<CommandProcessingResult<serde_json::Value, serde_json::Value, serde_json::Value>, GameEngineError>;

    fn get_info(&self) -> GameEngineInfo;
}

/// WASM-loaded game engine
pub struct WasmGameEngine {
    module: wasmtime::Module,
    store: wasmtime::Store<GameEngineState>,
}

/// gRPC-connected game engine
pub struct GrpcGameEngine {
    client: GameEngineServiceClient<Channel>,
    game_code: String,
}
```

### API Endpoints

```rust
/// Start a new game session
#[derive(Deserialize)]
pub struct StartSessionRequest {
    pub player_id: String,
    pub currency: String,
    pub operator_id: String,
    pub token: String,
}

#[derive(Serialize)]
pub struct StartSessionResponse {
    pub session_id: Uuid,
    pub public_state: serde_json::Value,
    pub balance: Decimal,
}

/// Execute a spin
#[derive(Deserialize)]
pub struct SpinRequest {
    pub session_id: Uuid,
    pub bet_amount: Decimal,
    pub bet_lines: Option<u32>,
}

#[derive(Serialize)]
pub struct SpinResponse {
    pub success: bool,
    pub outcome: serde_json::Value,
    pub public_state: serde_json::Value,
    pub win_amount: Decimal,
    pub balance: Decimal,
    pub bonus_triggered: Option<TriggeredBonus>,
}
```

### Wallet Integration

```rust
/// Operator wallet client
#[async_trait]
pub trait WalletClient: Send + Sync {
    /// Debit player balance (for bets)
    async fn debit(
        &self,
        player_id: &str,
        amount: Decimal,
        currency: &str,
        transaction_id: Uuid,
        game_round_id: Uuid,
    ) -> Result<WalletResponse, WalletError>;

    /// Credit player balance (for wins)
    async fn credit(
        &self,
        player_id: &str,
        amount: Decimal,
        currency: &str,
        transaction_id: Uuid,
        game_round_id: Uuid,
    ) -> Result<WalletResponse, WalletError>;

    /// Rollback a transaction
    async fn rollback(
        &self,
        transaction_id: Uuid,
    ) -> Result<(), WalletError>;

    /// Get player balance
    async fn get_balance(
        &self,
        player_id: &str,
        currency: &str,
    ) -> Result<Decimal, WalletError>;
}

pub struct WalletResponse {
    pub transaction_id: Uuid,
    pub balance: Decimal,
    pub currency: String,
}
```

### Audit Trail

```rust
/// Game round record for GLI-19 audit
pub struct GameRoundRecord {
    pub round_id: Uuid,
    pub session_id: Uuid,
    pub player_id: String,
    pub game_code: String,
    pub timestamp: DateTime<Utc>,
    pub bet_amount: Decimal,
    pub win_amount: Decimal,
    pub currency: String,
    pub command: GameActionCommand,
    pub rng_outcome_hash: String,  // SHA-256 of canonicalized RNG outcome
    pub public_state_before: serde_json::Value,
    pub public_state_after: serde_json::Value,
    pub outcome: serde_json::Value,
}

/// Audit service
#[async_trait]
pub trait AuditService: Send + Sync {
    async fn record_round(&self, record: GameRoundRecord) -> Result<(), AuditError>;
    async fn get_round(&self, round_id: Uuid) -> Result<Option<GameRoundRecord>, AuditError>;
    async fn get_session_rounds(&self, session_id: Uuid) -> Result<Vec<GameRoundRecord>, AuditError>;
    async fn verify_rng(&self, round_id: Uuid) -> Result<bool, AuditError>;
}
```

## Configuration

```toml
# config.toml
[server]
host = "0.0.0.0"
port = 8080
workers = 4

[redis]
url = "redis://redis:6379"
session_ttl_seconds = 3600

[rng]
grpc_url = "http://rng-service:50051"

[database]
url = "postgres://user:pass@postgres:5432/rgs"
max_connections = 20

[engines]
# WASM engines
[engines.happy-panda]
type = "wasm"
path = "/engines/happy-panda.wasm"
version = "1.0.0"

# gRPC engines
[engines.treasure-hunt]
type = "grpc"
url = "http://treasure-hunt-engine:50052"

[operators]
[operators.demo]
wallet_url = "http://demo-wallet:8081"
api_key = "${DEMO_WALLET_API_KEY}"
```

## Reference Files

| Rust Module | TypeScript Reference |
|-------------|---------------------|
| `src/api/routes.rs` | `omnitronix-rgs/src/controllers/` |
| `src/session/store.rs` | `omnitronix-rgs/src/services/session.service.ts` |
| `src/wallet/client.rs` | `omnitronix-rgs/src/services/wallet.service.ts` |
| `src/engine/registry.rs` | `omnitronix-rgs/src/services/game-engine.service.ts` |
| `src/audit/service.rs` | `omnitronix-rgs/src/services/audit.service.ts` |

## Features

- `wasm-engines` - Support loading game engines as WASM modules
- `grpc-engines` - Support connecting to game engines via gRPC
- `postgres` - PostgreSQL for audit trail and configuration
- `redis` - Redis for session storage
- `opensearch` - OpenSearch for analytics and audit search
- `metrics` - Prometheus metrics endpoint

## Deployment

```dockerfile
FROM rust:1.75 as builder
WORKDIR /app
COPY . .
RUN cargo build --release --features "wasm-engines,grpc-engines,postgres,redis"

FROM debian:bookworm-slim
COPY --from=builder /app/target/release/rust-rgs /usr/local/bin/
COPY --from=builder /app/config.toml /etc/rgs/
EXPOSE 8080
CMD ["rust-rgs", "--config", "/etc/rgs/config.toml"]
```

## Usage

```bash
# Run server
cargo run --release -- --config config.toml

# Health check
curl http://localhost:8080/api/v1/health

# Start session
curl -X POST http://localhost:8080/api/v1/games/happy-panda/session \
  -H "Content-Type: application/json" \
  -d '{"player_id": "player123", "currency": "EUR", "operator_id": "demo", "token": "..."}'

# Execute spin
curl -X POST http://localhost:8080/api/v1/games/happy-panda/spin \
  -H "Content-Type: application/json" \
  -d '{"session_id": "...", "bet_amount": "1.00"}'
```

## License

MIT
