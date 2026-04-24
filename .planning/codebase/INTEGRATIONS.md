# External Integrations

**Analysis Date:** 2026-04-24

## APIs & External Services

**Messaging:**
- Telegram Bot API - Receives commands and sends bot messages via long polling
  - SDK/Client: `aiogram==3.4.1` in `bot.py`
  - Auth: `DCA_TELEGRAM_BOT_TOKEN` from `.env` (validated in `bot.py`)

**Exchange/Trading API:**
- FixedFloat REST API (`https://ff.io/api/v2`) - Fetches pairs/limits and creates/tracks swap orders (`ccies`, `price`, `create`, `order`)
  - SDK/Client: `requests==2.31.0` via `ff_request`/`ff_request_async` in `bot.py`
  - Auth: `FF_API_KEY` and `FF_API_SECRET` (HMAC-SHA256 signature in `ff_sign` in `bot.py`)

**Blockchain RPC:**
- EVM JSON-RPC providers for Arbitrum, BSC, Polygon (mainnet/testnet) - Balance reads, allowance checks, gas estimation, and transaction broadcasting
  - SDK/Client: `web3==6.15.1` in `erc20.py` and `auto_send.py`
  - Auth: No API key configured in code; public RPC endpoints from `networks.py` and fallback lists in `erc20.py`

**Explorer/Reference Links:**
- Blockchair and chain explorers (Arbiscan/BscScan/PolygonScan) - Used for user-facing transaction links
  - SDK/Client: Direct URL composition in `networks.py` and `bot.py`
  - Auth: Not applicable

## Data Storage

**Databases:**
- SQLite (local file database)
  - Connection: `DATABASE_PATH` env var or default `dca.db` in project root (`bot.py`)
  - Client: `aiosqlite==0.19.0` in `bot.py`

**File Storage:**
- Local filesystem only
  - Keystore files in `keystores/` managed by `wallet.py`
  - Logs and runtime files in `logs/` (`bot.py`)
  - Optional bootstrap wallet input file `wallet.json` documented in `README.md` and `WALLET_SETUP.md`

**Caching:**
- None external
- In-memory process cache only (`_web3_cache`, `_balances_cache`, `CACHE_TTL`) in `bot.py`

## Authentication & Identity

**Auth Provider:**
- Custom access control (no third-party identity provider)
  - Implementation: Telegram sender identity plus `ADMIN_USER_ID` gate in `AccessControlMiddleware` (`bot.py`)

**Credential Storage:**
- OS keyring for wallet password storage via `keyring==24.3.0` (`wallet.py`)

## Monitoring & Observability

**Error Tracking:**
- None detected (no Sentry/Bugsnag/Rollbar integration)

**Logs:**
- Python `logging` to file and stdout in `bot.py`
- Default file path: `logs/bot.log`

## CI/CD & Deployment

**Hosting:**
- Self-hosted local runtime (manual process execution via `python3 bot.py` per `README.md`)

**CI Pipeline:**
- None detected (no `.github/workflows/`, no CI config files)

## Environment Configuration

**Required env vars:**
- `DCA_TELEGRAM_BOT_TOKEN` (`bot.py`, `.env.example`)
- `ADMIN_USER_ID` (`bot.py`, `.env.example`)
- `FF_API_KEY` (`bot.py`, `.env.example`)
- `FF_API_SECRET` (`bot.py`, `.env.example`)

**Optional env vars:**
- `DATABASE_PATH` (`bot.py`)
- `BOT_LOCK_PATH` (`bot.py`)
- `LAST_SEEN_EXECUTION_FILE` (`bot.py`)
- `DCA_EXECUTION_WINDOW_SECONDS` (`bot.py`)
- `USE_TESTNET` (`networks.py`, `bot.py`)

**Secrets location:**
- `.env` in project root (present in `.gitignore`)
- OS keyring entries (service `AutoDCA_Bot`) in `wallet.py`
- Encrypted keystore JSON files in `keystores/` (managed by `wallet.py`)

## Webhooks & Callbacks

**Incoming:**
- None - bot uses Telegram long polling (`dp.start_polling`) in `bot.py`

**Outgoing:**
- No webhook callbacks implemented
- Outbound HTTPS calls are direct API/RPC requests to FixedFloat and blockchain RPC endpoints from `bot.py` and `erc20.py`

---

*Integration audit: 2026-04-24*
