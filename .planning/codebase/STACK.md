# Technology Stack

**Analysis Date:** 2026-04-24

## Languages

**Primary:**
- Python 3.9+ - Main application logic in `bot.py`, `auto_send.py`, `erc20.py`, `wallet.py`, `networks.py`

**Secondary:**
- SQL (SQLite dialect) - Schema and queries embedded in `bot.py` (`init_db`, scheduler and transaction state queries)
- Markdown - Operational and setup docs in `README.md`, `WALLET_SETUP.md`, `TESTING.md`, `MIGRATION_GUIDE.md`

## Runtime

**Environment:**
- CPython 3.9.6 observed in `venv/pyvenv.cfg`
- Enforced minimum runtime is Python 3.9 in `bot.py`

**Package Manager:**
- pip 21.2.4 (system Python 3.9 environment)
- Dependency manifest: `requirements.txt`
- Lockfile: missing (no `poetry.lock`, `Pipfile.lock`, or `requirements.lock`)

## Frameworks

**Core:**
- aiogram 3.4.1 - Telegram bot framework and polling loop in `bot.py`
- web3 6.15.1 - EVM RPC client and transaction flow in `erc20.py` and `auto_send.py`

**Testing:**
- Not detected - no automated test runner config (`pytest`, `unittest`, `nose`, `tox`) in repository files
- Manual test-mode guidance exists in `TESTING.md`

**Build/Dev:**
- venv (stdlib) - local virtual environment workflow documented in `README.md`
- python-dotenv 1.0.0 - runtime env loading in `bot.py` and `networks.py`

## Key Dependencies

**Critical:**
- `aiogram==3.4.1` - Telegram command/update handling in `bot.py`
- `requests==2.31.0` - FixedFloat REST integration in `bot.py`
- `aiosqlite==0.19.0` - persistent DCA state and transaction tracking in `bot.py`
- `web3==6.15.1` - chain RPC, balances, gas, signing flow in `erc20.py` and `auto_send.py`
- `eth-account==0.10.0` - account/tx signing and keystore decryption in `wallet.py` and `erc20.py`

**Infrastructure:**
- `python-dotenv==1.0.0` - `.env` loading from project root in `bot.py` and `networks.py`
- `keyring==24.3.0` - OS keychain credential storage in `wallet.py`
- `eth-keyfile==0.6.0` and `cryptography==42.0.5` - Ethereum keystore format and encryption support for wallet operations in `wallet.py`

## Configuration

**Environment:**
- Runtime env vars are loaded from `.env` using `load_dotenv` in `bot.py` and `networks.py`
- Baseline variables are documented in `.env.example`
- Required secrets/config for startup are validated in `bot.py`: `DCA_TELEGRAM_BOT_TOKEN`, `ADMIN_USER_ID`, `FF_API_KEY`, `FF_API_SECRET`
- Optional runtime overrides in `bot.py` and `networks.py`: `DATABASE_PATH`, `BOT_LOCK_PATH`, `LAST_SEEN_EXECUTION_FILE`, `DCA_EXECUTION_WINDOW_SECONDS`, `USE_TESTNET`

**Build:**
- Dependencies installed from `requirements.txt`
- No dedicated build system or packaging config (`pyproject.toml`, `setup.py`, `setup.cfg`) detected

## Platform Requirements

**Development:**
- Local machine execution model (`README.md`): Windows, macOS, or Linux
- Python 3.9+ with pip and venv support
- Writable local filesystem for `logs/`, `keystores/`, and SQLite DB path (`bot.py`, `wallet.py`)
- Working OS keyring backend required for password persistence (`wallet.py`)

**Production:**
- Long-running local process (`python3 bot.py`) with continuous internet access to Telegram, FixedFloat, and RPC providers (`bot.py`, `erc20.py`, `networks.py`)
- Not containerized; no deployment manifests or orchestration configs detected in repo root

---

*Stack analysis: 2026-04-24*
