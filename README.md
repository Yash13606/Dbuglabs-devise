# Dbuglabs-devise

**Team ID: DBG-462** &nbsp;·&nbsp; HACKBACK · dBug Labs, solo lab

Five verified findings about **[accountill](https://github.com/Panshak/accountill)**, an open-source MERN invoicing app for freelancers (Express 4 + Mongoose 5 API, React 17 client). Every finding cites the exact file and line, and carries an honest confidence tag.

The deliverable is a single document: **[`accountill-notes.md`](./accountill-notes.md)**.

## Contents

- [The five claims](#the-five-claims)
- [Verify it yourself](#verify-it-yourself)
- [Evidence tags](#evidence-tags)
- [Scope and method](#scope-and-method)
- [Known limits](#known-limits)
- [Repository contents](#repository-contents)

## The five claims

Four are things the AI agent got right, and each was confirmed by opening the cited lines. The fifth is a claim the agent got **wrong**, which was corrected.

| # | Finding | Evidence | Status | Fix |
|---|---|---|---|---|
| 1 | **No server route has an auth check.** Routers are mounted with no middleware, so all 24 endpoints are open. The auth middleware also appears to be unused. | `server/index.js:32-35` | Confirmed (routes) / Likely (middleware unused: absence found by search) | Mount auth on the data routes, but keep `GET /invoices/:id` readable for invoice recipients. |
| 2 | **`GET /clients` returns every user's clients.** The list handler has no owner filter. | `server/controllers/clients.js:44-45` | Confirmed | Filter by the owner taken from the verified token. |
| 3 | **All PDFs share one file, and anyone can download it.** `/create-pdf` and `/send-pdf` overwrite `invoice.pdf`; `GET /fetch-pdf` serves it with no id or login. | `server/index.js:57,88,97-99` | Confirmed (lines) / Likely (cross-user swap under concurrent use) | Render to a buffer and return it in the same response; remove `/fetch-pdf`. |
| 4 | **Invoice numbers are `count + 1`, computed in the browser, with no unique index.** Deleting an invoice makes the next number repeat. | `client/src/components/Invoice/Invoice.js:93-96`, `server/models/InvoiceModel.js:13` | Confirmed | Server-side counter plus a unique index on `(creator, invoiceNumber)`. |
| 5 | **Corrected:** the receipt/invoice label is *not* wrong for balances of 1,000 or more. It is wrong only for an **overpayment of 1,000 or more**. | `client/src/utils/utils.js:2-4`, `server/documents/index.js:140` | Confirmed | Send `balanceDue` as a number and format it only for display. |

### About the correction (claim 5)

The agent said the label logic breaks above 1,000 because `Number("1,000")` is `NaN`. The code shows something narrower. `NaN <= 0` is `false`, so a comma-formatted *positive* balance still falls into the "Invoice" branch, which is the right answer. Only a negative balance of -1,000 or lower (an overpayment) gets the wrong label. The document has a table of sample values that shows each case.

## Verify it yourself

Run these from the root of the `accountill/` folder. Each command prints the lines the claims rely on.

```bash
# 1. Routers are mounted with no middleware, and nothing imports the auth middleware
sed -n 32,35p server/index.js
grep -rn "middleware/auth" server --include=*.js    # no output = no importer

# 2. The client list has no owner filter
sed -n 44,45p server/controllers/clients.js

# 3. One shared PDF file, written by two routes and served by a third
sed -n '57p;88p;97,99p' server/index.js

# 4. Invoice number = count + 1, and no unique index on it
sed -n '93p;96p' client/src/components/Invoice/Invoice.js
sed -n 13p server/models/InvoiceModel.js

# 5. The label logic and the comma formatting behind the correction
sed -n 2,4p client/src/utils/utils.js
sed -n 140p server/documents/index.js
node -e 'const f=v=>v.toString().replace(/\B(?=(\d{3})+(?!\d))/g,",");
for (const v of [0,1000,-1000]) console.log(v, JSON.stringify(f(v)), Number(f(v)))'
# expected: 0 "0" 0 / 1000 "1,000" NaN / -1000 "-1,000" NaN
```

## Evidence tags

| Tag | Meaning |
|---|---|
| `[Confirmed]` | The cited line was opened and it proves the claim. |
| `[Likely]` | Strong signs, but no single line proves it, or it depends on behaviour that was not run. Absence claims such as "no file imports X" are tagged this way. |
| `[Guess]` | No evidence. Never used as a fact. |

## Scope and method

- **What was audited:** the `accountill` source tree, with paths relative to its root. The compiled `client/build/` folder was ignored.
- **How:** a read-only review. Nothing was installed, built or started, and the original code was neither changed nor copied into this repository.
- **Process:** an AI agent did the first pass (stack, routes, data model, gaps). Each claim kept here was then checked against the source by hand, and the one wrong claim was corrected.

## Known limits

- This is a static read. No request was sent to a running server, so runtime behaviour (for example two users rendering PDFs at the same moment in claim 3) is marked `[Likely]`, not proven.
- "Nothing imports the auth middleware" comes from a search of `server/`, which is why it is `[Likely]`.
- Dependency advisories were not checked.

## Repository contents

| File | Purpose |
|---|---|
| [`accountill-notes.md`](./accountill-notes.md) | The five claims in the playbook's Evidence / `[Confirmed]` format, each with the gap it feeds and the smallest fix. |
| `README.md` | This overview. |
