# Testing Patterns

**Analysis Date:** 2026-04-24

## Test Framework

**Runner:**
- Not detected (no `pytest`, `unittest`, `nose`, `tox` configuration files in repository root).
- Config: Not detected.

**Assertion Library:**
- Not detected (no automated assertion library in use).

**Run Commands:**
```bash
python3 bot.py          # Runtime smoke/integration execution (macOS/Linux, from `README.md`)
python bot.py           # Runtime smoke/integration execution (Windows, from `README.md`)
Not detected            # Watch mode
Not detected            # Coverage
```

## Test File Organization

**Location:**
- No automated test directory detected (`tests/` not present, no `test_*.py` files).
- Existing testing guidance is documentation-based in `TESTING.md` (project root).

**Naming:**
- Automated test naming pattern: Not detected.
- Runtime test-mode naming uses env flags referenced in `bot.py` and documented in `TESTING.md`:
  - `DRY_RUN`
  - `MOCK_FIXEDFLOAT`
  - `USE_TESTNET`

**Structure:**
```text
No test source tree detected
root/
  TESTING.md        # Manual testing playbook
  bot.py            # Runtime toggles and mode checks
  auto_send.py      # Dry-run behavior for transfer flow
  erc20.py          # Dry-run transaction preparation
```

## Test Structure

**Suite Organization:**
```python
# Pattern in `bot.py`: optional local test config import with runtime fallback
try:
    from test_config import DRY_RUN, MOCK_FIXEDFLOAT, USE_TESTNET, is_test_mode
except ImportError:
    DRY_RUN = False
    MOCK_FIXEDFLOAT = False
    USE_TESTNET = False
```

**Patterns:**
- Setup pattern: Start bot with env toggles, then exercise Telegram commands (`/setwallet`, `/setdca`, `/execute`, `/status`) as integration checks (`README.md`, `TESTING.md`, `bot.py`).
- Teardown pattern: Manual stop/restart process; no automated fixture teardown logic detected.
- Assertion pattern: Operational assertions are log-driven and state-driven (DB state transitions and transaction statuses in `bot.py`), not framework assertions.

## Mocking

**Framework:** Custom runtime mocking (no library-based mocking framework detected)

**Patterns:**
```python
# `bot.py`: mock API path during ff_request
if MOCK_FIXEDFLOAT:
    if method == "ccies":
        return get_mock_fixedfloat_ccies()["data"]
    elif method == "price":
        return get_mock_fixedfloat_price(network_key)["data"]
    elif method == "create":
        return get_mock_fixedfloat_order(network_key, amount, btc_address)["data"]
```

```python
# `erc20.py` and `auto_send.py`: blockchain dry-run path
if dry_run:
    logger.info("[DRY RUN] ...")
    return None
```

**What to Mock:**
- FixedFloat HTTP API interactions through `MOCK_FIXEDFLOAT` in `bot.py`.
- On-chain broadcasting through `dry_run` propagation (`bot.py` -> `auto_send.py` -> `erc20.py`).

**What NOT to Mock:**
- Keystore and keyring integration when validating real runtime behavior (`wallet.py`, `bot.py` startup flow).
- SQLite state transitions when validating scheduler/idempotency behavior (`bot.py` DB flows).

## Fixtures and Factories

**Test Data:**
```python
# Expected mock factory hooks in optional `test_config.py` (imported in `bot.py`)
get_mock_fixedfloat_order(...)
get_mock_fixedfloat_ccies(...)
get_mock_fixedfloat_price(...)
```

**Location:**
- `test_config.py` is referenced by `bot.py` but not present in repository.
- No committed fixture/factory directory detected.

## Coverage

**Requirements:** None enforced (no coverage config or CI coverage gate detected).

**View Coverage:**
```bash
Not detected
```

## Test Types

**Unit Tests:**
- Not used in repository state (no test runner and no unit test files).

**Integration Tests:**
- Manual integration testing is the active pattern:
  - API behavior toggled by `MOCK_FIXEDFLOAT` in `bot.py`.
  - Transaction behavior toggled by `DRY_RUN` and `USE_TESTNET` in `bot.py`, `networks.py`, `auto_send.py`, `erc20.py`.
  - Startup logging explicitly reports enabled test modes in `main()` inside `bot.py`.

**E2E Tests:**
- Not used (no automated end-to-end framework detected).

## Common Patterns

**Async Testing:**
```python
# Production async call pattern used for manual validation
success, approve_tx, transfer_tx, error_msg = await auto_send_usdt(
    network_key=from_asset,
    user_id=user_id,
    wallet_password=wallet_password,
    deposit_address=deposit_address,
    required_amount=required_amount,
    btc_address=btc_address,
    order_id=order_id,
    dry_run=DRY_RUN,
)
```

**Error Testing:**
```python
# Pattern used in runtime checks (`bot.py`)
if is_retryable_network_error(error_msg):
    # mark blocked/tx_pending and retry later
else:
    # mark failed and advance schedule
```

---

*Testing analysis: 2026-04-24*
