---
name: zm
description: ZenMoney personal finance CLI. Use when the user asks about transactions, expenses, accounts, tags/categories, wants to review or categorize transactions, check spending, or manage personal finances. Trigger phrases: "покажи траты", "добавь транзакцию", "категоризируй", "zm", "финансы", "транзакции", "ZenMoney", "расходы", "доходы".
---

# ZenMoney CLI (`zm`)

Personal finance management tool that talks to the ZenMoney API.

## Key facts

- **Binary:** `zm` (globally available via `~/.local/bin/zm`)
- **Data directory:** `/Volumes/dev/Projects/zerro_claude/zerro AI control/data/`
- **Amounts:** in currency units (EUR/RUB), NOT kopecks
- **IDs:** shown as 12-char prefixes in list output; commands accept 4+ char prefixes
- **Output:** denormalized JSON to stdout; status/errors to stderr
- **Mutations are local until pushed:** edit → `zm pending` → `zm push`

## Common workflows

### View transactions
```bash
zm list --limit 20
zm list --from 2026-01-01 --to 2026-03-31
zm list --last 12m           # last 12 full months
zm list --last 6m            # last 6 full months
zm list --last 1y            # last full calendar year
zm list --last ytd           # year-to-date
zm list --payee Prisma
zm list --untagged --type expense
zm list --tag "Продукты" --account "Revolut EUR Joint"
zm list --min 100 --max 500
zm list --in-balance         # only transactions from inBalance accounts
zm show <id-prefix>          # full details of one or more transactions
```

### Tag / categorize
```bash
zm tag-set <id> --tags "Продукты"          # replace all tags
zm tag-add <id> --tags "Кафе"              # add tag (appended → project role)
zm tag-add <id> --tags "Продукты" --as-category  # insert at front → category role
zm tag-add <id> --tags "Турция" --as-project     # append at end → project role
zm tag-remove <id> --tags "Продукты"       # remove tag
zm review --from 2026-02-01                # history-based suggestions
zm review --from 2026-02-01 --ai           # + AI for unknown payees
zm review --from 2026-02-01 --ai --apply   # interactive apply
```

### Create transaction
```bash
zm create --date 2026-04-10 --account "Revolut EUR" --outcome 42.50 \
  --payee "Lidl" --tags "Продукты"
```

### Edit / delete
```bash
zm edit <id> payee="New Name" comment="note" tags="Продукты,Кафе"
zm delete <id>
zm merge <id1> <id2>                        # sum amounts into first
```

### Transfers
```bash
zm make-transfer <expense-id> <income-id>       # pair → transfer
zm convert-to-transfer <id> --to "Savings"      # expense → transfer
zm convert-to-transfer <id> --from "Savings"    # income → transfer
```

### Bulk edits
```bash
echo '{"id":"c557","tags":["Приложения"],"comment":"domain"}' | zm batch
echo '...' | zm batch --dry-run
```

### Manage changes
```bash
zm pending             # show uncommitted changes
zm undo                # revert last change
zm undo <id-prefix>    # revert changes for specific transaction
zm undo --all          # revert all changes
zm push --dry-run      # preview
zm push                # send to ZenMoney API
```

### Expense / income report
```bash
zm report --last 12m                        # last 12 full months, inBalance, grouped by category
zm report --last 12m --group-by project     # group by project (last tag)
zm report --last 12m --group-by month       # monthly breakdown
zm report --last 12m --monthly              # category groups + per-month sub-breakdown
zm report --last 12m --project "Турция"     # drill-down: only that project, grouped by category
zm report --from 2025-01-01 --to 2025-12-31 --type income
zm report --last 12m --all-accounts         # include accounts not in inBalance
```

Output fields: `period`, `baseCurrency`, `accounts`, `totalTransactions`, `total`, `groups[]`.  
Each group: `name`, `type` (category/project/month), `total`, `avgPerMonth`, `pctOfTotal`, optionally `months`.

### Sync
```bash
zm sync                # incremental sync
zm sync --full         # full re-fetch
```

### Reference data
```bash
zm accounts            # list accounts with IDs and inBalance status
zm tags                # list tags/categories with IDs
```

## Command reference

| Command | Purpose |
|---------|---------|
| `sync` | Sync from ZenMoney API |
| `list` | List transactions (filterable) |
| `show` | Full details of transaction(s) |
| `accounts` | List accounts |
| `tags` | List tags/categories |
| `tag-create` | Create a new tag |
| `create` | Create a new transaction |
| `pending` | Show uncommitted local changes |
| `tag-set` | Replace tags on transaction(s) |
| `tag-add` | Add tags to transaction(s) |
| `tag-remove` | Remove tags from transaction(s) |
| `edit` | Edit transaction fields |
| `merge` | Merge two transactions (sum amounts) |
| `make-transfer` | Convert expense+income pair into transfer |
| `convert-to-transfer` | Convert standalone expense/income into transfer |
| `delete` | Soft-delete transaction(s) |
| `undo` | Revert pending change(s) |
| `batch` | Bulk edits from stdin (JSON-lines) |
| `review` | Suggest tags based on history (optionally AI-powered) |
| `report` | Aggregate expense/income report by category/project/month |
| `push` | Push pending changes to API |

## Output fields

`zm list` and `zm show` output includes:
- `amount` — human-readable string with original currency (e.g. `-4000.00 RUB`, `+94.33 EUR`)
- `amountEur` — numeric EUR equivalent, ready for aggregation (e.g. `-40.00`, `-94.33`)
- `category` — first tag (category role); `"(без категории)"` if no tags
- `project` — last tag (project role); `null` if 0–1 tags
- `tags` — full tag list (all tags including middle ones for 3+)

**For aggregation always use `amountEur`**, not `amount`. Parsing the `amount` string and summing across currencies gives wrong results (e.g. 4000 RUB would be treated as 4000 EUR).

**Tag roles:** first tag = category, last tag = project. Use `category`/`project` for grouping in reports to avoid double-counting. Use `--as-category` / `--as-project` with `tag-add` to control insertion position.

**`--in-balance` filter:** restricts results to transactions from accounts with `inBalance=True` (checks `outcomeAccount` for expenses, `incomeAccount` for income). Includes archived accounts. Use for "real money" reporting.

**`--last` shortcut:** `--last 12m` = last 12 full months, `--last 1y` = last full calendar year, `--last ytd` = year-to-date. Mutually exclusive with `--from`/`--to`.

## Behavior guidelines

- Always show `zm list` output before making changes so the user can confirm the right transaction
- After tagging or editing, show `zm pending` to confirm what will be pushed
- Don't `zm push` without explicit user confirmation
- When unsure about a tag name, run `zm tags` first
- Use `--dry-run` on `push` and `batch` when the user wants to preview
