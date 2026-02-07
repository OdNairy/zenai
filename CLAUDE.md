# CLAUDE.md — ZenMoney AI Control

## Overview

CLI tool (`zm`) for managing ZenMoney transactions via API. Single Python file, no external dependencies.

## How to Run

```bash
# Sync data from ZenMoney API
./zm sync              # incremental
./zm sync --full       # full re-fetch

# Query transactions
./zm list --limit 20
./zm list --from 2026-01-01 --to 2026-01-31 --payee Prisma
./zm list --untagged --type expense
./zm list --tag Продукты --account "Revolut EUR Joint"
./zm list --min 100 --max 500

# Show details
./zm show <id-prefix>
./zm accounts
./zm tags

# Modify transactions (creates local pending changes)
./zm tag-add <id> --tags "Продукты"
./zm tag-set <id1> <id2> --tags "Кафе" "Досуг"
./zm tag-remove <id> --tags "Продукты"
./zm edit <id> payee="New Payee" comment="note"
./zm delete <id>
./zm merge <id1> <id2>              # sum amounts into first
./zm make-transfer <expense> <income>  # convert pair to transfer

# Manage changes
./zm pending           # show uncommitted changes
./zm undo              # revert last change
./zm undo --all        # revert all changes
./zm push --dry-run    # preview API push
./zm push              # send to ZenMoney API
```

## Architecture

- **Single file:** `zm` (Python 3, no dependencies beyond stdlib)
- **Data dir:** `data/` (gitignored) — contains `token`, `raw.json`, `state.json`, `pending.json`
- All output is **denormalized JSON to stdout**, errors/status to stderr
- IDs accept UUID prefixes (min 4 chars); ambiguous prefix → error with matches
- Transaction amounts are in **currency units** (not kopecks)
- Tags are `null` or `["uuid1", ...]` arrays

## Key Classes

| Class | Purpose |
|-------|---------|
| `DataStore` | File I/O for raw.json, pending.json, state.json |
| `Lookup` | Build maps: accounts_by_id, tags_by_id/name, instruments, merchants |
| `Formatter` | Denormalize transactions/accounts/tags for JSON output |
| `ApiClient` | HTTP POST to ZenMoney `/v8/diff/` via urllib |

## Mutation Flow

1. Write commands append to `pending.json` with old/new field values
2. `zm list`/`zm show` overlay pending changes on raw data
3. `zm push` merges pending by (entity, id), POSTs to API, updates raw.json, clears pending
4. `zm undo` removes last (or all) pending entries

## ZenMoney API

- Endpoint: `https://api.zenmoney.ru/v8/diff/`
- Auth: Bearer token in `data/token`
- Single POST endpoint for both read (sync) and write (push)
- Request: `{"currentClientTimestamp": N, "serverTimestamp": N, ...entities}`
- Response: updated entities + new serverTimestamp
