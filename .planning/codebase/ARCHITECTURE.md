# Architecture

**Analysis Date:** 2026-04-24

## Pattern Overview

**Overall:** Modular monolith centered on a single asynchronous Telegram bot process.

**Key Characteristics:**
- One runtime process starts in `bot.py`, then owns command handling, background loops, and lifecycle.
- Feature logic is grouped by responsibility across modules (`bot.py`, `auto_send.py`, `erc20.py`, `wallet.py`, `networks.py`) without package-level layering.
- Durable workflow state is persisted in SQLite (`dca_plans`, `sent_transactions`, `wallets`, `completed_orders`) and used as an execution state machine.

## Layers

**Interface Layer (Telegram Adapter):**
- Purpose: Receive Telegram commands and return user-facing responses.
- Location: `bot.py`
- Contains: Aiogram dispatcher setup, middleware, and handlers like `/start`, `/setdca`, `/execute`, `/status`, `/walletstatus`.
- Depends on: Orchestration/database functions in `bot.py`, wallet/blockchain helpers in `wallet.py`, `auto_send.py`, `erc20.py`.
- Used by: Runtime polling loop (`dp.start_polling(bot)` in `bot.py`).

**Application Orchestration Layer:**
- Purpose: Coordinate DCA lifecycle, idempotency, retries, order progression, and scheduling.
- Location: `bot.py`
- Contains: `dca_scheduler`, `order_monitor`, `cmd_execute`, claim/release helpers, expiry handling, order status polling, FixedFloat order creation.
- Depends on: Persistence in `bot.py` DB calls, external clients (`ff_request*`, `auto_send_usdt`, `get_web3_instance`), configuration (`networks.py`).
- Used by: Telegram handlers and background tasks started from `main` in `bot.py`.

**Persistence Layer:**
- Purpose: Store and recover plans, wallet metadata, transaction lifecycle, and completion history.
- Location: `bot.py`, `wallet.py`
- Contains: SQLite schema/init/migrations in `init_db` (`bot.py`), query/update paths in command handlers and schedulers (`bot.py`), keystore files in `keystores/` managed by `wallet.py`.
- Depends on: `aiosqlite` (`bot.py`), JSON filesystem + OS keyring (`wallet.py`).
- Used by: Scheduler, monitor, manual execution, wallet setup/deletion/status flows.

**Blockchain Transaction Layer:**
- Purpose: EVM RPC connectivity, USDT balance/allowance checks, gas estimation, approve/transfer execution.
- Location: `erc20.py`, `auto_send.py`
- Contains: RPC fallback client creation (`erc20.py`), ERC20 methods (`approve_usdt`, `transfer_usdt`, `check_allowance` in `erc20.py`), end-to-end guarded send flow (`auto_send_usdt` in `auto_send.py`).
- Depends on: Network definitions in `networks.py`, keystore decryption in `wallet.py`, `web3` and `eth-account`.
- Used by: `cmd_execute` and `dca_scheduler` paths in `bot.py`.

**External Exchange API Layer:**
- Purpose: Interact with FixedFloat for network discovery, limits, order creation, and order status/txid lookup.
- Location: `bot.py`
- Contains: Signed API client (`ff_sign`, `ff_request`, `ff_request_async`), limits/network mapping (`get_fixedfloat_limits`, `update_network_codes`), order creation (`create_fixedfloat_order`).
- Depends on: Env credentials loaded in `bot.py`, `requests`, and bot-level retry/error mapping logic.
- Used by: `/limits`, `/setdca`, `/execute`, `dca_scheduler`, `order_monitor` in `bot.py`.

**Configuration Layer:**
- Purpose: Provide chain/network metadata and runtime path/env resolution.
- Location: `networks.py`, `bot.py`
- Contains: Static chain configs and helpers in `networks.py`; bot env/path/runtime checks in `bot.py`.
- Depends on: `.env` loading via `python-dotenv`.
- Used by: All blockchain and orchestration flows.

## Data Flow

**Scheduled DCA Auto Execution (`dca_scheduler`):**

1. Background loop in `bot.py` (`dca_scheduler`) reads due active plans from `dca_plans`.
2. For each plan, `bot.py` resolves in-flight/active order state using `sent_transactions` + FixedFloat status checks.
3. `bot.py` validates limits through `get_fixedfloat_limits` and atomically claims execution (`execution_state` in `dca_plans`).
4. `bot.py` creates a FixedFloat order (`create_fixedfloat_order`) and stores active order fields in `dca_plans`.
5. If wallet/password exists, `bot.py` inserts `sent_transactions` state `sending` and calls `auto_send_usdt` in `auto_send.py`.
6. `auto_send.py` decrypts wallet via `wallet.py`, checks balances via `erc20.py`, performs approve/transfer, and returns tx hashes/state.
7. `bot.py` updates `sent_transactions` state (`sent`, `tx_pending`, `blocked`, `failed`) and advances `next_run` in `dca_plans` based on result policy.

**Manual Execution (`/execute`):**

1. Command handler `cmd_execute` in `bot.py` parses target plan and loads plan/order state from `dca_plans`.
2. If an in-flight/active order exists, `bot.py` reconciles state (resume transfer, pending check, fail/confirm transitions) before allowing new order creation.
3. For new orders, `bot.py` reuses the same order creation + optional auto-send path as scheduler (FixedFloat + `auto_send_usdt`).
4. Results are persisted into `sent_transactions` and reflected to user messages from `bot.py`.

**Order Completion Monitoring (`order_monitor`):**

1. Background loop in `bot.py` (`order_monitor`) scans active orders and sent tx records.
2. `bot.py` checks FixedFloat order status and marks completion/failure via `mark_order_completed` / `mark_order_failed`.
3. On completion, `bot.py` fetches BTC txid and updates `completed_orders` + Telegram progress message tracking.

**Wallet Provisioning (`/setwallet`):**

1. `cmd_setwallet` in `bot.py` reads `wallet.json` from project root.
2. `bot.py` creates encrypted keystore and saves it through `save_keystore` in `wallet.py` to `keystores/`.
3. `bot.py` stores password in OS keyring via `wallet.py`, caches it in `_wallet_passwords`, and upserts wallet metadata into `wallets`.

**State Management:**
- Durable state: SQLite tables in `bot.py` (`dca_plans`, `wallets`, `sent_transactions`, `completed_orders`).
- Runtime state: in-memory caches in `bot.py` (`_wallet_passwords`, `_web3_cache`, `_balances_cache`, `_order_progress_messages`).
- Concurrency controls: DB claim fields (`execution_state`, state transitions in `sent_transactions`) and per-wallet async send lock in `auto_send.py`.

## Key Abstractions

**DCA Plan:**
- Purpose: Recurring purchase schedule and current active exchange order context.
- Examples: `dca_plans` schema/init and updates in `bot.py`.
- Pattern: Single row per plan with mutable `next_run` and active order fields (`active_order_*`).

**Transaction Lifecycle Record:**
- Purpose: Track on-chain send pipeline and recover after restart.
- Examples: `sent_transactions` schema and state transitions in `bot.py`.
- Pattern: Explicit state machine (`sending`, `approve_confirmed`, `transfering`, `tx_pending`, `blocked`, `sent`, `confirmed`, `failed`, `expired`).

**Wallet Identity + Secret Material:**
- Purpose: Maintain per-user EVM wallet address and decrypt capability without storing plaintext key in DB.
- Examples: `wallet.py` (`save_keystore`, `decrypt_private_key`, keyring helpers), wallet table interactions in `bot.py`.
- Pattern: Keystore file + OS keyring secret + DB address metadata.

**Network Descriptor:**
- Purpose: Normalize chain-specific parameters for all on-chain operations.
- Examples: `NETWORKS_*` and getters in `networks.py`.
- Pattern: Keyed config dictionary (`USDT-ARB`, `USDT-BSC`, `USDT-POLYGON`) consumed across modules.

## Entry Points

**Process Entry Point:**
- Location: `bot.py`
- Triggers: Python process start (`python bot.py`).
- Responsibilities: Runtime checks, lock acquisition, DB init, password preload, network code refresh, recovery, background task spawn, Telegram polling.

**Telegram Command Endpoints:**
- Location: `bot.py`
- Triggers: Aiogram decorators `@dp.message(...)`.
- Responsibilities: User command API for planning (`/setdca`), wallet management (`/setwallet`, `/deletewallet`, `/walletstatus`), execution (`/execute`), controls (`/pause`, `/resume`, `/delete`), and observability (`/status`, `/history`, `/limits`, `/ping`).

**Background Scheduler:**
- Location: `bot.py` (`dca_scheduler`)
- Triggers: `asyncio.create_task(dca_scheduler())` in `main`.
- Responsibilities: Periodic due-plan execution and retry/skip orchestration.

**Background Order Monitor:**
- Location: `bot.py` (`order_monitor`)
- Triggers: `asyncio.create_task(order_monitor())` in `main`.
- Responsibilities: Poll exchange completion state and finalize/notify outcomes.

## Error Handling

**Strategy:** Localized try/except boundaries with stateful recovery and retry-aware branching.

**Patterns:**
- Loop-guarded recovery: scheduler/monitor loops in `bot.py` catch per-plan and top-level exceptions, then continue.
- Retry classification: `bot.py` maps retryable network errors vs non-retryable and sets transaction states accordingly.
- Idempotent resumption: `bot.py` and `auto_send.py` resume from stored tx hashes/states after restart or pending transitions.
- External call hardening: `erc20.py` retries RPC connection/tx send with fallback providers; `bot.py` wraps FixedFloat HTTP errors into runtime errors.

## Cross-Cutting Concerns

**Logging:** Structured logging initialized in `bot.py`; all modules use `logging.getLogger(__name__)`.
**Validation:** Input/env/runtime validation in `bot.py` (command arg parsing, BTC/network checks, startup prerequisites) and address/key checks in `auto_send.py` / `wallet.py`.
**Authentication:** Single-admin access control middleware in `bot.py` (`AccessControlMiddleware`) gates all bot interactions by `ADMIN_USER_ID`.

---

*Architecture analysis: 2026-04-24*
