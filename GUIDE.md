# ZenMoney CLI (`zm`) — AI Agent Guide

You have access to `./zm`, a CLI for reading and modifying ZenMoney transactions. All output is JSON to stdout, status/errors to stderr. Always run from the project directory.

## Before anything: sync data
```
./zm sync
```

## Reading data

```bash
./zm list [--from YYYY-MM-DD] [--to YYYY-MM-DD] [--payee TEXT] [--tag NAME] [--account NAME] [--type expense|income|transfer] [--untagged] [--min N] [--max N] [--limit N]
./zm show <id-prefix> [<id2>...]  # full detail for one or more transactions (4+ char prefix)
./zm accounts [--all]       # list accounts (--all includes archived)
./zm tags [--flat]          # list categories
```

Default limit is 50. Filters are substring matches (case-insensitive for payee/account/tag).

## Creating transactions

```bash
./zm create --account "Account Name" --outcome 123.45   # expense
./zm create --account "Account Name" --income 500.00    # income
# Optional: --date YYYY-MM-DD (default: today), --payee TEXT, --tags "T1" "T2", --comment TEXT
```

## Modifying transactions

All writes are local until pushed. Each command accepts one or more 4+ char ID prefixes.

```bash
./zm tag-set <ids...> --tags "Tag1" "Tag2"     # replace all tags
./zm tag-add <ids...> --tags "Tag1"             # add keeping existing
./zm tag-remove <ids...> --tags "Tag1"          # remove specific (omit --tags to clear all)
./zm edit <id> payee="Name" comment="text" date=YYYY-MM-DD outcome=123.45
./zm delete <ids...>                            # soft-delete
./zm merge <id1> <id2> [id3...]                 # sum amounts into first, delete rest
./zm make-transfer <expense-id> <income-id>     # combine into one transfer
```

## Managing pending changes

```bash
./zm pending             # list local changes not yet pushed
./zm undo                # revert last change
./zm undo --all          # revert everything
./zm push --dry-run      # preview what would be sent
./zm push                # send to ZenMoney API
```

## Output format

List items look like:
```json
{"id": "571e1176", "date": "2026-02-07", "type": "expense", "amount": "-23.11 EUR", "account": "Revolut EUR Joint", "payee": "Prisma Olari", "tags": ["Продукты"], "comment": null}
```

Transfers show `"from"` / `"to"` instead of `"account"`, and multi-currency amounts as `"55.34 USD → 140.00 BYN"`.

## Typical workflows

**Tag untagged transactions in a date range:**
1. `./zm list --untagged --from 2026-01-01 --to 2026-01-31` — find them
2. `./zm tag-add <id1> <id2> --tags "Кафе"` — assign tags
3. `./zm push` — send to server

**Find and fix a payee name:**
1. `./zm list --payee "old name"` — find transactions
2. `./zm edit <id> payee="Correct Name"` — fix each
3. `./zm push`

**Convert matched expense+income into a transfer:**
1. `./zm list --from 2026-01-15 --to 2026-01-15 --type expense` — find the expense
2. `./zm list --from 2026-01-15 --to 2026-01-15 --type income` — find the matching income
3. `./zm make-transfer <expense-id> <income-id>`
4. `./zm push`

## Key facts
- Amounts are in currency units (23.11 EUR), not kopecks
- Tags are arrays of UUIDs; the tool resolves names automatically
- Tag names support partial matching (unique substring works)
- `--dry-run` on push is always safe; `push` modifies the server
- `undo` only reverts local pending, not already-pushed changes
