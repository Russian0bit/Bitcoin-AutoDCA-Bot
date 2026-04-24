# Codebase Concerns

**Analysis Date:** 2026-04-24

## Tech Debt

**Monolithic bot orchestration:**
- Issue: Trading logic, API integration, scheduler loops, DB migrations, wallet management, and Telegram handlers all live in one file and one module-level runtime context.
- Files: `bot.py`
- Impact: High change risk, hard reviewability, and frequent regression surface because unrelated concerns are tightly coupled.
- Fix approach: Split into bounded modules (`services/scheduler`, `services/fixedfloat`, `repositories/sqlite`, `handlers/telegram`, `wallet/security`) and move shared state behind explicit interfaces.

**Duplicated order execution flows:**
- Issue: Scheduler execution and manual `/execute` flow duplicate large blocks for order creation, active-order persistence, auto-send, and state updates.
- Files: `bot.py`
- Impact: Bug fixes must be applied in multiple places; drift in behavior is likely.
- Fix approach: Extract a single `execute_plan(plan_id, trigger)` service and reuse from both scheduler and command handlers.

**Stringly-typed state machine:**
- Issue: Transaction/order lifecycle uses raw strings (`sending`, `transfering`, `blocked`, `tx_pending`, `confirmed`, `failed`, `expired`) in many branches.
- Files: `bot.py`
- Impact: Typos and invalid transitions are easy to introduce; impossible to enforce legal transitions statically.
- Fix approach: Define centralized enum/constants plus transition helpers with validation.

**Data model precision debt:**
- Issue: Financial amounts are stored and compared as `float` values in Python and SQLite (`REAL`).
- Files: `bot.py`, `erc20.py`, `auto_send.py`
- Impact: Rounding drift can affect duplicate detection, limit checks, and transfer amount fidelity.
- Fix approach: Use `Decimal` in application code and store normalized integer units (minor units/wei) in DB.

**Runtime schema migration in application startup:**
- Issue: DDL and data migration logic execute inside `init_db()` at runtime without migration versioning.
- Files: `bot.py`
- Impact: Startup becomes riskier; partial migrations can block bot startup or leave inconsistent schema states.
- Fix approach: Introduce versioned migrations (Alembic-style or custom migration table) and run them as explicit admin operations.

## Known Bugs

**Duplicate cleanup keeps oldest row, not latest state:**
- Symptoms: Startup cleanup removes duplicate `sent_transactions` rows by keeping `MIN(id)` per `order_id`.
- Files: `bot.py`
- Trigger: Existing duplicate rows for the same `order_id` on startup.
- Workaround: Manually inspect `sent_transactions` before startup and preserve the newest state row.

**Declared BTC validation in auto-send is not implemented:**
- Symptoms: `auto_send_usdt()` documents BTC address validation but does not validate or use `btc_address`.
- Files: `auto_send.py`
- Trigger: Any call to `auto_send_usdt(...)`.
- Workaround: Validate BTC address before calling `auto_send_usdt` and remove/repurpose unused argument until implemented.

**Testnet/network support mismatch:**
- Symptoms: `NETWORK_CODES` includes `USDT-POLYGON`, while testnet network config excludes Polygon; configured plans can pass validation and fail later on network resolution.
- Files: `bot.py`, `networks.py`
- Trigger: Running with `USE_TESTNET=true` and using Polygon-related flows.
- Workaround: Restrict allowed assets dynamically from `networks.NETWORKS` when testnet mode is enabled.

**BTC address validator accepts format-only addresses and rejects testnet formats:**
- Symptoms: Validation is regex-only; checksum is not verified and testnet `tb1`/`m`/`n` addresses are rejected.
- Files: `bot.py`
- Trigger: `/setdca` address validation.
- Workaround: Replace regex-only validation with library-based checksum validation and network-aware prefixes.

**Blockchair link helper ignores selected network mode:**
- Symptoms: `get_blockchair_url()` always returns mainnet URL regardless of `USE_TESTNET` and per-network config.
- Files: `networks.py`
- Trigger: Any caller expecting testnet-aware explorer links.
- Workaround: Build URL from `get_network_config(...)[\"blockchair_base\"]` or global mode-aware mapping.

## Security Considerations

**Wallet passwords cached in-process memory:**
- Risk: Wallet passwords are loaded into `_wallet_passwords` and persist for process lifetime.
- Files: `bot.py`
- Current mitigation: Password source of truth is OS keyring; cache is cleared on `/deletewallet`.
- Recommendations: Minimize cache lifetime, fetch on demand from keyring, and apply explicit memory scrubbing strategy where possible.

**Plain `wallet.json` ingest path in project root:**
- Risk: Private key/password are read from a filesystem file in repo root during `/setwallet`; local file exposure risk depends on host permissions/backup tooling.
- Files: `bot.py`
- Current mitigation: File is overwritten to a keystore-only JSON after setup.
- Recommendations: Enforce restrictive permissions before read, support one-time stdin/secure prompt import, and avoid writing secrets into project-root files.

**No explicit SQLite foreign key enforcement on connections:**
- Risk: Declared foreign keys are not guaranteed unless `PRAGMA foreign_keys=ON` is set per connection.
- Files: `bot.py`
- Current mitigation: Foreign keys are toggled during one migration block only.
- Recommendations: Enable foreign keys for every DB connection immediately after connect.

**Environment override behavior can unexpectedly trust local `.env`:**
- Risk: `load_dotenv(..., override=True)` allows `.env` values to override process environment values.
- Files: `bot.py`, `networks.py`
- Current mitigation: Required env vars are validated at startup.
- Recommendations: Use `override=False` in production and separate local-dev vs production config loaders.

## Performance Bottlenecks

**Scheduler/monitor perform high-latency operations in long sequential loops:**
- Problem: FixedFloat status checks, RPC checks, and DB updates run in nested loops with limited batching.
- Files: `bot.py`
- Cause: Monolithic control flow and per-item network calls.
- Improvement path: Split phases (fetch, classify, persist), batch DB writes, and apply bounded concurrency for independent orders.

**Frequent open/close of SQLite connections in hot paths:**
- Problem: Many short-lived `aiosqlite.connect()` calls inside loops and helpers.
- Files: `bot.py`
- Cause: No repository/session abstraction.
- Improvement path: Reuse connection per loop cycle or use pooled access abstraction; reduce commit frequency for related writes.

**Missing indexes for common query predicates:**
- Problem: Queries repeatedly filter by `plan_id`, `state`, `active`, `deleted`, `next_run`, and `sent_at`, but only `order_id` unique index is created.
- Files: `bot.py`
- Cause: Incomplete indexing strategy in `init_db()`.
- Improvement path: Add indexes like `(active, deleted, next_run)`, `(plan_id, state, sent_at)`, `(plan_id, order_id)`.

**Wallet status triggers repeated RPC reads:**
- Problem: Balance checks may query same values multiple times (`chain_id`, `block_number`, balance retries) for each network request.
- Files: `bot.py`
- Cause: Defensive retry logic duplicates expensive RPC calls.
- Improvement path: Centralize per-network health probe with cached probe results per command execution.

## Fragile Areas

**Order/transaction lifecycle branching:**
- Files: `bot.py`
- Why fragile: Complex branching over order status + tx status + expiry + retryability appears in scheduler, recovery, monitor, and `/execute`.
- Safe modification: Change transition logic in one centralized state-transition module and reuse everywhere.
- Test coverage: No automated tests cover transition matrix.

**Startup migration and cleanup path:**
- Files: `bot.py`
- Why fragile: Runtime DDL + data cleanup + index creation + commit/rollback handling sit in one startup function.
- Safe modification: Isolate migrations into explicit versions and add dry-run/verifier tooling before applying.
- Test coverage: No migration tests or rollback-path tests.

**Command parsing via prefix matching and bare `except`:**
- Files: `bot.py`
- Why fragile: `/execute`, `/pause`, `/resume`, `/delete` rely on manual parsing and broad `except` blocks that silently swallow parse errors.
- Safe modification: Use explicit command argument parsing utilities and narrow exception handling.
- Test coverage: No parser tests for malformed command inputs.

**Runtime caches with global mutable state:**
- Files: `bot.py`, `auto_send.py`
- Why fragile: `_wallet_passwords`, `_web3_cache`, `_balances_cache`, `_SEND_LOCKS`, `_order_progress_messages` are mutable globals shared across tasks.
- Safe modification: Encapsulate caches in services with lifecycle controls and explicit cache invalidation rules.
- Test coverage: No concurrency/race tests around cache mutation.

## Scaling Limits

**Single-user gate:**
- Current capacity: One allowed Telegram user (`ADMIN_USER_ID`).
- Limit: Multi-tenant usage is blocked by middleware design.
- Scaling path: Move to per-user authorization model and tenant-scoped configuration.

**Single-process runtime with local lock file:**
- Current capacity: One bot process per shared DB/lock path.
- Limit: No horizontal workers or distributed scheduling.
- Scaling path: Externalize scheduler state/locks (Redis/Postgres advisory locks) and run stateless workers.

**SQLite + local filesystem persistence:**
- Current capacity: Suitable for low write concurrency and local deployment.
- Limit: Write contention and operational limits under multi-instance or high-frequency activity.
- Scaling path: Move to network DB (Postgres) and managed secret storage.

**Plan volume and latency coupling:**
- Current capacity: Per-user plan count is capped at 3 per network in command logic.
- Limit: Sequential scheduler/monitor flow latency grows with number of active plans/orders.
- Scaling path: Queue-based execution and bounded concurrent workers with idempotent jobs.

## Dependencies at Risk

**No lockfile / hash-pinned dependency set:**
- Risk: Reproducibility and supply-chain consistency are weak across environments.
- Impact: Runtime drift and hard-to-reproduce issues.
- Migration plan: Introduce lockfile (`pip-tools`/`uv`/Poetry) with hash-checking installs.

**`web3` + middleware compatibility surface:**
- Risk: Middleware API changes are handled with fallback imports and broad exception catches.
- Impact: Chain connectivity can fail at runtime after dependency changes.
- Migration plan: Pin compatible ranges, add integration tests per supported network, and fail fast with explicit diagnostics.

**Synchronous HTTP dependency in async app path:**
- Risk: `requests` remains in critical API path (`ff_request`) and relies on `to_thread` wrappers.
- Impact: Thread-pool pressure and hidden blocking behavior under load.
- Migration plan: Move FixedFloat client to native async HTTP (`aiohttp`/`httpx`) with centralized retries/backoff.

## Missing Critical Features

**Automated test suite and CI checks:**
- Problem: No automated unit/integration tests or CI gating are present.
- Blocks: Safe refactoring of scheduler/state logic and migration logic.

**Idempotency guard for order creation at API boundary:**
- Problem: Internal DB claim guards exist, but no explicit external idempotency key is attached to FixedFloat create calls.
- Blocks: Strong protection against duplicate external orders during retry/restart edge cases.

**Structured observability for critical workflows:**
- Problem: Logging is file/console-based without metrics, alerting, or health endpoints.
- Blocks: Fast detection of stuck states (`blocked`, `tx_pending`, `claiming`) in unattended operation.

## Test Coverage Gaps

**Order-state transition matrix is untested:**
- What's not tested: Paths across `sending`/`approve_confirmed`/`transfering`/`tx_pending`/`blocked`/`failed`/`confirmed`/`expired`.
- Files: `bot.py`, `auto_send.py`
- Risk: Silent regressions in retry/idempotency handling.
- Priority: High

**Migration and startup recovery are untested:**
- What's not tested: `init_db()` migration branches, duplicate cleanup behavior, pending-tx recovery, stale claim recovery.
- Files: `bot.py`
- Risk: Startup can corrupt or misclassify persisted execution state.
- Priority: High

**Wallet security and keyring edge cases are untested:**
- What's not tested: `/setwallet` malformed input, keyring backend failures, delete-wallet cleanup guarantees.
- Files: `bot.py`, `wallet.py`
- Risk: Partial setup/teardown leaves inconsistent auth/send state.
- Priority: Medium

**Network validation correctness is untested:**
- What's not tested: BTC address validator correctness, testnet address handling, network/testnet compatibility rules.
- Files: `bot.py`, `networks.py`
- Risk: User accepts invalid targets or cannot configure valid testnet scenarios.
- Priority: Medium

---

*Concerns audit: 2026-04-24*
