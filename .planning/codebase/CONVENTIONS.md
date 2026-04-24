# Coding Conventions

**Analysis Date:** 2026-04-24

## Naming Patterns

**Files:**
- Use lowercase snake_case module names in project root: `bot.py`, `auto_send.py`, `erc20.py`, `wallet.py`, `networks.py`.

**Functions:**
- Use snake_case for sync and async functions: `create_web3_client` in `erc20.py`, `auto_send_usdt` in `auto_send.py`, `cmd_setdca` in `bot.py`.
- Prefix private helpers with leading underscore: `_is_retryable_send_error` in `erc20.py`, `_extract_amount_from_error` in `bot.py`.
- Prefix Telegram command handlers with `cmd_`: `cmd_start`, `cmd_execute`, `cmd_walletstatus` in `bot.py`.

**Variables:**
- Use snake_case for locals and module variables: `wallet_address`, `min_native_required`, `existing_order_id`.
- Use UPPER_SNAKE_CASE for constants and config: `MIN_PYTHON_VERSION` in `bot.py`, `GAS_MULTIPLIER` in `erc20.py`, `NETWORKS_MAINNET` in `networks.py`.

**Types:**
- Use `PascalCase` for classes: `AccessControlMiddleware` in `bot.py`.
- Use built-in generics and `typing` hints together:
  - Built-in: `tuple[str, float]`, `dict[str, str]` in `bot.py`.
  - `typing`: `Optional[str]`, `Dict[str, Any]` in `bot.py`, `wallet.py`.

## Code Style

**Formatting:**
- Tool used: Not detected (`black`, `ruff format`, `yapf`, `isort` config files not present).
- Key settings: Not detected; style is manual and consistent with 4-space indentation and frequent type annotations in `bot.py`, `erc20.py`, `auto_send.py`, `wallet.py`, `networks.py`.
- String formatting pattern is f-string heavy across modules (for logs and user messages), for example in `bot.py` and `auto_send.py`.

**Linting:**
- Tool used: Not detected (`ruff`, `flake8`, `pylint`, `mypy` configs not present).
- Key rules: Not detected.

## Import Organization

**Order:**
1. Standard library imports first (e.g., `asyncio`, `os`, `time` in `bot.py` and `auto_send.py`).
2. Third-party imports second (e.g., `aiogram`, `aiosqlite`, `web3`, `requests`, `keyring` in `bot.py`, `erc20.py`, `wallet.py`).
3. Local module imports last (e.g., `from networks import ...`, `from wallet import ...` in `bot.py`, `auto_send.py`, `erc20.py`).

**Path Aliases:**
- Not used. Imports are direct module-relative by filename (`from networks import ...`, `from erc20 import ...`).

## Error Handling

**Patterns:**
- Convert dependency/import failures into explicit startup `RuntimeError` with install guidance in `bot.py` (top-level guarded imports).
- Validate environment early and fail fast (`ADMIN_USER_ID`, `DCA_TELEGRAM_BOT_TOKEN`) in `bot.py`.
- Use `try/except` at IO boundaries and network boundaries:
  - Web3 RPC calls and transaction send/retry in `erc20.py`.
  - Auto-send orchestration and receipt polling in `auto_send.py`.
  - Scheduler and command handler fallbacks in `bot.py`.
- Use typed/semantic exception translation:
  - Raise `ValueError` for invalid input/state (`networks.py`, `wallet.py`, `bot.py`).
  - Raise `RuntimeError` for operational failures (`erc20.py`, `bot.py`).
- Broad `except Exception` is common in orchestration paths; expected pattern is "log + return safe fallback + continue loop" in `bot.py` and `auto_send.py`.

## Logging

**Framework:** `logging`

**Patterns:**
- Root logging configured once in `bot.py` via `logging.basicConfig(...)` with:
  - file handler to `logs/bot.log`
  - stream handler to stdout/stderr
  - level `INFO`
- Per-module loggers via `logger = logging.getLogger(__name__)` in `bot.py`, `auto_send.py`, `erc20.py`, `wallet.py`.
- Severity usage pattern:
  - `logger.info` for lifecycle/progress.
  - `logger.warning` for recoverable conditions.
  - `logger.error` for failed operations.
- Sensitive value masking pattern exists for wallet/address display in `auto_send.py` and `erc20.py`.

## Comments

**When to Comment:**
- Use module-level docstrings to describe responsibility (`wallet.py`, `networks.py`, `erc20.py`, `auto_send.py`).
- Use section banners for large flow segmentation in `bot.py`.
- Use inline comments to explain defensive behavior or migration intent (e.g., DB schema migration in `bot.py`, PoA fallback in `erc20.py`).

**JSDoc/TSDoc:**
- Not applicable (Python codebase).
- Python docstrings are used for most public functions and many internal helpers.

## Function Design

**Size:** 
- Utility modules keep mostly short-to-medium functions (`wallet.py`, `networks.py`, `erc20.py` helpers).
- `bot.py` contains very large orchestration functions (`dca_scheduler`, command handlers) alongside utility helpers.

**Parameters:**
- Prefer explicit typed parameters.
- Frequently pass primitive values instead of custom classes (e.g., `network_key`, `plan_id`, `user_id`, `order_id` across `bot.py` and `erc20.py`).

**Return Values:**
- Use explicit tuple returns for multi-outcome operations:
  - `auto_send_usdt` returns `(success, approve_tx_hash, transfer_tx_hash, error_message)` in `auto_send.py`.
- Use `Optional[...]` for nullable lookups (`load_keystore`, `load_password_from_keyring` in `wallet.py`).

## Module Design

**Exports:**
- Modules expose plain functions/constants directly; no explicit export list (`__all__`) in `bot.py`, `auto_send.py`, `erc20.py`, `wallet.py`, `networks.py`.
- `bot.py` acts as composition root and runtime entrypoint (`if __name__ == "__main__": asyncio.run(main())`).

**Barrel Files:**
- Not used.

---

*Convention analysis: 2026-04-24*
