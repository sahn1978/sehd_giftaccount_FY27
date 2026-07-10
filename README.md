# Gift & Program Fund Ledger

A small static dashboard for tracking gift and program fund balances over
time: what's left, what's come in, what's been spent, and whether each fund
is on pace to be spent down by **May 31, 2027**.

It's built to match how the data actually arrives: the business office
sends a pivot export (fund names down the side, a total balance in the
column), and that export repeats every week or so with a new date. Rather
than maintaining one growing master file, this project just takes a new
CSV file each time and drops it into the `data` folder. The dashboard finds
every CSV sitting there on its own and combines them.

Live view: once this repo is pushed to GitHub with Pages turned on, the
dashboard is at `https://<your-username>.github.io/<repo-name>/`.

## How the pieces fit together

```
gift-dashboard/
  index.html                 the whole dashboard (HTML + CSS + JS in one file)
  _headers                   Netlify Basic Auth config, see "Password protection" below
  data/
    2026-06-30.csv             one export's worth of fund balances
    2026-07-10.csv             another export, dropped in as its own file
  scripts/
    append_business_office_export.py   turns a new export into a new dated CSV
  README.md
```

`index.html` runs entirely in the visitor's browser. On load, it asks
GitHub's API "what files are in the `data` folder right now," fetches every
`.csv` it finds there, and combines them before drawing anything. That
means updating the dashboard is just: drop a new CSV into `data`, commit,
push. GitHub Pages re-serves the new file and the next visitor sees it,
with no build step and nothing to merge by hand.

## The file format

Each CSV in `data` is a small table, one row per fund. The columns are:

| Column | Meaning |
|---|---|
| `pull_date` | The date this row is as of |
| `transaction_date` | The date of a specific gift or expense, if known. Leave blank for reconciliation rows |
| `fund_code` | Short fund identifier, e.g. `BG002185` |
| `fund_name` | Full fund name, e.g. `Spad Program-406820` |
| `category` | `Gift`, `Program`, or similar, parsed from the business office's label |
| `entry_type` | One of `Balance Reconciliation`, `New Gift`, `Expense`, `Adjustment` |
| `amount` | Positive number, only used for `New Gift` / `Expense` / `Adjustment` rows |
| `running_balance_reported` | The exact total the business office reported for that fund on that date. Only used on `Balance Reconciliation` rows |
| `description` | Free text, optional |
| `source` | Where the row came from - e.g. `Business Office Export`, or `Manual Entry` |

Two kinds of rows do two different jobs:

- **Balance Reconciliation** rows are the backbone. They carry the exact
  number from the business office pivot, in the business office's own sign
  convention: **negative means the fund still has money available; positive
  means the fund has been overspent.** This is what the "available balance
  over time" chart is built from, and it's always trustworthy because it
  comes straight from the source.
- **New Gift** and **Expense** rows are optional, itemized entries you can
  add any time you know something more specific than the weekly total -
  for example, "a $25,000 pledge payment landed on the 13th." These show up
  in the fund register as "logged gifts" and "logged expenses," separate
  from the reconciled balance.

Dates in the `pull_date` and `transaction_date` columns can be written
either way - `2026-07-17` or `7/17/2026` both work.

**A file can hold just that week's rows, or your full history to date -
either works.** Some weeks you might only have a handful of new rows to
add; other times it's easiest to just re-save everything you have so far
into one new file. The dashboard deduplicates automatically: if the same
fund's balance for the same date, or the same logged gift, shows up in more
than one file, it's only counted once. You never need to worry about
double counting by re-including old rows in a new file.

## Weekly update workflow

1. Get the new pivot export from the business office (same shape as
   before: `Driver`, `Total amount (date)`).
2. Convert it to a CSV:
   ```
   pip install openpyxl
   python3 scripts/append_business_office_export.py path/to/new_export.xlsx
   ```
   This reads the export, splits each `Driver` into fund code / name /
   category, pulls the date from the column header, and writes a new file
   named `data/<date>.csv`. If the date can't be read from the column
   header, pass it directly:
   ```
   python3 scripts/append_business_office_export.py new_export.xlsx --date 2026-07-17
   ```
3. (Optional) If you know about a specific gift or expense that hasn't
   shown up in a reconciliation yet, you can add it as an extra row in
   that same CSV, or as its own small CSV, with `entry_type` set to
   `New Gift` or `Expense` and a positive `amount`.
4. Add the new file to your repository the same way you did the first
   time: open the `data` folder on GitHub, use **Add file → Upload
   files**, drop in the new CSV, and commit. The page updates on its own
   once GitHub Pages redeploys, usually within a minute or two.

## Publishing on GitHub Pages

1. Create a new repository and push this folder to it.
2. In the repo, go to **Settings → Pages**.
3. Under **Build and deployment**, set **Source** to `Deploy from a branch`,
   branch `main`, folder `/ (root)`.
4. Save. GitHub gives you a URL a minute later -
   `https://<your-username>.github.io/<repo-name>/`.
5. Share that link with the Dean. No login is needed to view it beyond the
   password screen described below, so keep in mind the fund balances are
   visible to anyone with the link unless the repository is private
   (GitHub Pages on a private repo requires GitHub Enterprise or a paid
   plan to stay private - check what your account supports if that matters
   here).

A technical note: the dashboard figures out which GitHub repository to ask
for files by reading its own URL, which only works cleanly on a standard
`https://username.github.io/repo-name/` address. If you ever move this to
a custom domain, that auto-detection would need to be set manually instead.

## Password protection

The page shows a password screen before the dashboard loads. It's worth
understanding exactly what this does and doesn't do, since this project
holds real fund data.

**What the built in screen actually is.** It's a light deterrent, not real
security. Everything that ships to a visitor's browser, including the
password check itself, can be read by opening the page's source or the
browser's developer tools. More importantly, the CSV files in `data/` are
plain public files once this is on GitHub Pages. Anyone who knows or
guesses the address can fetch them directly, whether or not they ever see
the password screen. Treat this the way you'd treat a "don't share this
link" request: fine for keeping the dashboard from turning up in casual
browsing or search engines, not something to rely on for data that truly
needs to stay confidential.

**Changing the password.** Generate a new hash and paste it into
`index.html` in place of the `PASSWORD_HASH` value:
```
python3 -c "import hashlib; print(hashlib.sha256('yourpassword'.encode()).hexdigest())"
```
The plain password itself is never stored in the file, only this hash.

**If the data genuinely needs to stay private,** GitHub Pages can't do
that on its own; a real login wall for a Pages site requires GitHub
Enterprise Cloud, aimed at large organizations rather than a single
dashboard like this. A free option that actually works is to host the
exact same files on Netlify instead of GitHub Pages. This repo already
includes a `_headers` file with the settings Netlify needs: open it,
replace `dean:changeme` with your own username and password, then connect
this GitHub repository to a new Netlify site (netlify.com, "Add new site"
→ "Import an existing project" → pick this repo, no build command needed,
publish directory is the repo root). Netlify asks visitors for that
username and password before serving anything, including the CSV files,
which is real, server enforced protection rather than a cosmetic screen.
This works on Netlify's free plan.

## Notes on the pace/deadline calculation

For each fund, the dashboard compares its available balance at the first
export against the latest one to get a simple average weekly burn rate,
then projects forward to see whether the fund would hit zero before or
after May 31, 2027. This is a straight-line estimate, not a forecast - a
fund with only one or two exports so far will have a rough estimate that
firms up as more weekly data comes in. Funds that are currently overspent,
or that haven't shown any net spending yet, are flagged for review rather
than given a projected date.
