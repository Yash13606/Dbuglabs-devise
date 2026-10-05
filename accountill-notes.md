# accountill-notes.md

Solo lab: **accountill** (MERN invoicing app). 5 claims in OBSERVATIONS format.
Four claims the agents got right and I confirmed by opening the lines (a fact that is an absence found by search, such as "no file imports X", is tagged `[Likely]`, not `[Confirmed]`). One claim an agent got wrong, which I corrected (claim 5).

Tags: `[Confirmed]` I opened the line and it proves the claim. `[Likely]` strong signs, no single line proves it. `[Guess]` no evidence, never used as fact.
Read-only: nothing was run, installed or changed in the original repo. All paths are relative to `accountill/`.

---

## Claim 1: every server route is open, because no route has an auth check

- No server route is wrapped in any auth check: the routers are mounted, and every route is registered with a handler only. So all 24 server endpoints can be called by anyone.
  Evidence: `server/index.js:32-35` (routers mounted with no middleware), `server/routes/clients.js:6-10` and `server/routes/invoices.js:6-11` (handlers only) [Confirmed]
- The auth middleware is defined but no file imports it. This is an absence found by searching `server/`, not something one line proves, so it is only `[Likely]`.
  Evidence: `server/middleware/auth.js:7` (defined); a search of `server/` for `middleware/auth` found no importer [Likely]
- The client does send `Authorization: Bearer <token>`, but the server never reads it.
  Evidence: `client/src/api/index.js:6-12` [Confirmed]

**Gap this feeds:** anyone can read, change or delete every user's invoices, clients and profiles. Fix: mount auth on the data routes, but keep `GET /invoices/:id` readable, because invoice recipients open that link.

## Claim 2: `GET /clients` returns every user's clients

- The list handler counts and fetches clients with no filter, so one request returns all owners' client names, emails, phones and addresses, 8 per page.
  Evidence: `server/controllers/clients.js:44` (`countDocuments({})`), `server/controllers/clients.js:45` (`find()` with no condition) [Confirmed]
- The route has no middleware in front of it.
  Evidence: `server/routes/clients.js:6` [Confirmed]

**Gap this feeds:** client contact data of all freelancers is exposed. Fix: filter by the owner taken from the verified token.

## Claim 3: all PDFs go through one shared file that anyone can download

- Both `/create-pdf` and `/send-pdf` render to the same relative file `invoice.pdf`, so each render overwrites the previous user's PDF.
  Evidence: `server/index.js:57` (`/send-pdf` writes it), `server/index.js:88` (`/create-pdf` writes it) [Confirmed]
- `GET /fetch-pdf` returns that file to any caller, with no id and no login.
  Evidence: `server/index.js:97-99` [Confirmed]
- The client downloads in two separate calls (create, then fetch), so the file it gets back can belong to someone else's render.
  Evidence: `client/src/components/InvoiceDetails/InvoiceDetails.js:121` (create), `:140` (fetch) [Confirmed that the two calls exist; that a concurrent render swaps the file is [Likely]]

**Gap this feeds:** the last invoice rendered by any user (client name, address, line items, amounts) can be read by anyone. Fix: render to a buffer and return it in the same response, then delete `/fetch-pdf`.

## Claim 4: invoice numbers are `count + 1`, computed in the browser, with no unique constraint

- The client asks for the user's invoice count and uses count + 1 as the next number.
  Evidence: `client/src/components/Invoice/Invoice.js:93` (fetches the count), `:96` (sets `invoiceNumber` to `count + 1`, zero-padded) [Confirmed]
- The model has no unique index on the number, so duplicates are not rejected. Deleting an earlier invoice makes the next number repeat.
  Evidence: `server/models/InvoiceModel.js:13` (`invoiceNumber: String`, no `unique`) [Confirmed]
- The only unique indexes in the whole data model are the two `email` fields.
  Evidence: `server/models/userModel.js:5`, `server/models/ProfileModel.js:5` [Confirmed]

**Gap this feeds:** duplicate or colliding invoice numbers. Fix: server-side counter plus a unique index on `(creator, invoiceNumber)`.

## Claim 5: CORRECTED. The receipt/invoice label is **not** broken for amounts of 1,000 or more

**What the agent said (recon agent 1c):** `balanceDue` is sent as a comma-formatted string, and `Number("1,000")` is `NaN`, so the Receipt-vs-Invoice branch "is only correct for values below 1,000".

**What the code actually does:**
- The client formats the balance with commas before sending it.
  Evidence: `client/src/utils/utils.js:2-4` (`toCommas`), `client/src/components/InvoiceDetails/InvoiceDetails.js:137` and `:169` (`balanceDue: toCommas(total - totalAmountReceived)`) [Confirmed]
- The templates decide the label with `Number(balanceDue) <= 0`. `NaN <= 0` is `false`, so any comma-formatted value gets the **invoice** label.
  Evidence: `server/documents/index.js:140`, `server/documents/email.js:131` and `:136` [Confirmed]
- I checked the arithmetic by running the one-line regex on sample values (not the target's code):

  | balance sent | string after `toCommas` | `Number()` | label shown | correct? |
  |---|---|---|---|---|
  | `0` | `"0"` | 0 | Receipt | yes |
  | `500` | `"500"` | 500 | Invoice | yes |
  | `1000` | `"1,000"` | NaN | Invoice | **yes** |
  | `25000` | `"25,000"` | NaN | Invoice | **yes** |
  | `-500` | `"-500"` | -500 | Receipt | yes |
  | `-1000` | `"-1,000"` | NaN | Invoice | **no** (overpaid, should be Receipt) |

**Corrected claim:** a large positive balance still gets the correct "Invoice" label, because `NaN` falls into the same branch as a positive number. The label is wrong only when a customer has **overpaid by 1,000 or more** (a negative balance of -1,000 or lower). It is a narrow edge-case bug, not a "wrong above 1,000" bug. The agent overstated it. [Confirmed]

**Fix:** send `balanceDue` as a plain number and format it only for display, or strip commas before `Number()`.
