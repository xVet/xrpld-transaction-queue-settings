# Recommended transaction queue settings for xrpld

**Applies to:** networked hubs, validators and other xrpld nodes on mainnet. Verified against xrpld 3.4.1.

## 1. Check the current state

```shell
xrpld fee
grep -A8 '^\[transaction_queue\]' /etc/xrpld/xrpld.cfg
```

In the `fee` output, note `expected_ledger_size`, `current_queue_size` and `max_queue_size`.

## 2. Update xrpld.cfg

```ini
[transaction_queue]
minimum_txn_in_ledger = 256
target_txn_in_ledger = 256
maximum_txn_in_ledger = 500
minimum_txn_in_ledger_standalone = 500
slow_consensus_decrease_percent = 75
```

- **Replace** any existing `[transaction_queue]` section. Keep only one: if the file has two, xrpld merges them and the last value for each key wins.
- Put the section header on a line of its own, and use plain whole numbers (no commas, decimals or units). xrpld silently ignores values it cannot read.
- `minimum_txn_in_ledger_standalone = 500` is required. Its default is 1000, and xrpld will not start if `maximum_txn_in_ledger` is lower than it. It is only used in standalone mode.
- `maximum_txn_in_ledger = 500` caps `expected_ledger_size` at 500. Once the open ledger holds more transactions than `expected_ledger_size`, the fee needed rises to about 500 times the base fee and keeps rising as the open ledger fills.
- This is a local config change only. No amendment is involved.

## 3. Restart

```shell
sudo systemctl restart xrpld
```

Restart one server at a time. Wait until `server_state` is `full` (`proposing` for validators) before moving to the next. If xrpld does not start, check the error with:

```shell
journalctl -u xrpld -n 50 --no-pager
```

## 4. Verify

| Check | Expected |
|---|---|
| `xrpld fee` -> `expected_ledger_size` | between 256 and 500 |
| `xrpld fee` -> `current_queue_size` during busy periods | rises, then drains within a few ledgers |
| `server_info` -> `load_factor_fee_escalation` | absent at normal load, present only while fees are escalated |
| `server_info` -> `validated_ledger.age` | stays at a few seconds |

A value of `expected_ledger_size` below 256 means the section was not read. A value above 500 means the cap is not active.

## Notes

- Users are unaffected at normal load. While fees are escalated, low-fee transactions are queued or must pay the current open ledger fee (`open_ledger_fee` in the `fee` output, for a standard transaction).
- If `current_queue_size` stays above zero outside busy periods while `expected_ledger_size` reads 500, consider raising the cap.
