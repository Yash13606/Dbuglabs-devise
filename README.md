# Dbuglabs-devise

**Team ID: DBG-462**

Solo-lab submission for **HACKBACK · dBug Labs**: a reverse-engineering write-up of **accountill**, an open-source MERN invoicing app for freelancers ([original repo](https://github.com/Panshak/accountill)).

The only deliverable in this repo is [`accountill-notes.md`](./accountill-notes.md).

## What the document is

`accountill-notes.md` holds **5 verified claims** about the accountill codebase, written in the playbook's evidence format:

```
- <claim>
  Evidence: path/to/file:line [Confirmed]
```

Four claims are things the AI agent got right, and I confirmed each one by opening the cited line. The fifth is a claim the agent got **wrong**, which I corrected.

| # | Claim | Where to look |
|---|---|---|
| 1 | Every server route is open. The auth middleware exists but is never imported or mounted. | `server/index.js:32-35` |
| 2 | `GET /clients` returns every user's clients, with no owner filter. | `server/controllers/clients.js:44-45` |
| 3 | All PDFs are written to one shared `invoice.pdf`, and `GET /fetch-pdf` serves it to anyone. | `server/index.js:57,88,97-99` |
| 4 | Invoice numbers are `count + 1`, computed in the browser, with no unique index. | `client/src/components/Invoice/Invoice.js:93-96` |
| 5 | **Corrected:** the receipt/invoice label is *not* wrong for balances of 1,000 or more. It is wrong only for overpayments of 1,000 or more. | `client/src/utils/utils.js:2-4`, `server/documents/index.js:140` |

Each claim also says which gap it feeds and the smallest fix.

## The corrected claim (claim 5)

The agent said the label logic breaks above 1,000 because `Number("1,000")` is `NaN`. The code shows something narrower. `NaN <= 0` is false, so a comma-formatted positive balance still falls into the "Invoice" branch, which is the right answer. Only a negative balance of -1,000 or lower, an overpayment, gets the wrong label. The document includes a table of sample values that shows this.

## How to read the evidence tags

- `[Confirmed]`: I opened that line and it proves the claim.
- `[Likely]`: strong signs, but no single line proves it, or it depends on runtime behaviour I did not run.
- `[Guess]`: no evidence. Never used as fact.

## How it was made

- The original repo was **read, not run**: nothing was installed, built or executed, and the original code was not changed or copied into this repo.
- An AI agent did the first pass (stack, routes, data model, gaps). Every claim kept here was then checked against the source by hand, and the one wrong claim was fixed.
- Paths in the document are relative to the original `accountill/` folder.
