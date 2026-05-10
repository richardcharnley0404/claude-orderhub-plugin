# Sales Metrics — Range Parsing

The user-supplied range argument is parsed into an ISO date pair `{ from, to }`. For current-period inputs (`today`, `7d`, `30d`, `mtd`, `qtd`, `ytd`) `to` is the end of today in the user's locale. For past-period inputs (`yesterday`, `last week`, `last month`) `to` is the end of that past window — see the table below for exact values.

## Recognised inputs

| Input | from | to |
|---|---|---|
| `today` | start of today | end of today |
| `yesterday` | start of yesterday | end of yesterday |
| `7d` (default) | 6 days before today | end of today |
| `30d` | 29 days before today | end of today |
| `mtd` | first day of current month | end of today |
| `qtd` | first day of current quarter | end of today |
| `ytd` | first day of current year | end of today |
| `last week` | Monday of previous ISO week | Sunday of previous ISO week |
| `last month` | first day of previous month | last day of previous month |

## Comparison ranges

For the delta metric, the prior period is the immediately preceding window of equal length. E.g. if `from = 2026-05-04`, `to = 2026-05-10` (7d), prior is `2026-04-27` to `2026-05-03`.

## Garbled input

If the input doesn't match any recognised form: assume `7d` and render a one-line note `(interpreted "<input>" as 7d)` above the revenue card.
