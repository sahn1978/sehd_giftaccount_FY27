# Gift & Program Fund Ledger

A small static dashboard for tracking gift and program fund balances over
time: what's left, what's come in, what's been spent, and whether each fund
is on pace to be spent down by **May 31, 2027**.

It's built to match how the data actually arrives: the business office sends
a pivot export (fund names down the side, a total balance in the column),
and that export repeats every week or so with a new date. Instead of
overwriting last week's numbers, this project keeps every export as its own
row in a growing CSV file, so the site can chart the change over time.

Live view: once this repo is pushed to GitHub with Pages turned on, the
dashboard is at `https://<your-username>.github.io/<repo-name>/`.

## How the pieces fit together

```
gift-dashboard/
  index.html                 the whole dashboard (HTML + CSS + JS in one file)
  data/
    ledger.csv                the database - one row per fund per export, ever
  scripts/
    append_business_office_export.py   turns a new export into new ledger rows
  README.md
```

`index.html` fetches `data/ledger.csv` in the visitor's browser and does all
the charting client-side (via Chart.js and PapaParse, loaded from a CDN).
There is no build step and no server. That means updating the dashboard is
just: edit the CSV, commit, push. GitHub Pages re-serves the new file and
the next visitor sees the update.

## The ledger schema

`data/ledger.csv` is a long-format table - a true database in the sense that
every row is a permanent, dated entry. Nothing is ever overwritten; new
information is always a new row appended to the bottom.

| Column | Meaning |
|---|---|
| `pull_date` | The date this row was added to the ledger (the export or entry date) |
| `transaction_date` | The actual date of a gift or expense, if known. Leave blank for reconciliation rows |
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
  means the fund has been overspent.** Every time you get a new export, its
  numbers become a new batch of these rows, dated with that export's date.
  This is what the "available balance over time" chart is built from, and
  it's always trustworthy because it comes straight from the source.

- **New Gift** and **Expense** rows are optional, itemized entries you can
  add any time you know something more specific than the weekly total - for
  example, "a $25,000 pledge payment landed on the 13th" or "we paid a
  $1,200 honorarium." These show up in the fund register as "logged gifts"
  and "logged expenses," clearly separate from the reconciled balance, so
  the dashboard never mixes an estimate with a confirmed number.

Because the reconciliation rows alone are enough to chart "left, spent,
new money" over time, you don't need itemized detail every week - just add
it when you happen to know it. The dashboard is designed to work either way.

## Weekly update workflow

1. Get the new pivot export from the business office (same shape as before:
   `Driver`, `Total amount (date)`).
2. From the repo folder, run:
   ```
   pip install openpyxl
   python3 scripts/append_business_office_export.py path/to/new_export.xlsx
   ```
   This reads the export, splits each `Driver` into fund code / name /
   category, pulls the date from the column header, and appends one
   `Balance Reconciliation` row per fund to `data/ledger.csv`. It's safe to
   re-run - it skips a date it's already loaded unless you pass `--force`.
   If the date can't be read from the column header, pass it directly:
   ```
   python3 scripts/append_business_office_export.py new_export.xlsx --date 2026-07-17
   ```
3. (Optional) If you know about a specific gift or expense that hasn't shown
   up in a reconciliation yet, add a row to `data/ledger.csv` by hand -
   open it in Excel or a text editor, add one line with `entry_type` set to
   `New Gift` or `Expense` and a positive `amount`, and save as CSV (keep
   UTF-8 encoding).
4. Commit and push:
   ```
   git add data/ledger.csv
   git commit -m "Add July 17 export"
   git push
   ```
5. GitHub Pages rebuilds automatically. Refresh the live page in a minute
   or two and the new export is on the charts.

## Publishing on GitHub Pages

1. Create a new repository and push this folder to it.
2. In the repo, go to **Settings → Pages**.
3. Under **Build and deployment**, set **Source** to `Deploy from a branch`,
   branch `main`, folder `/ (root)`.
4. Save. GitHub gives you a URL a minute later -
   `https://<your-username>.github.io/<repo-name>/`.
5. Share that link with the Dean. No login is needed to view it, so keep in
   mind the fund balances are visible to anyone with the link unless the
   repository is private (GitHub Pages on a private repo requires GitHub
   Enterprise or a paid plan to stay private - check what your account
   supports if that matters here).

## About the mock data

`data/ledger.csv` currently holds two snapshots and one logged gift, built to
match where things actually stand right now rather than a long fabricated
history:

- **June 30, 2026** is a fabricated baseline for all seven funds, included
  so the chart has something to compare against. It's a guess, not a real
  export, since no real business office file from that date was available.
- **July 10, 2026** is your real, accurate export, the same numbers from
  the file you uploaded, entered exactly as reported.
- One **New Gift** row logs a $15,000 pledge payment landing in the Spad
  Program fund between those two dates, which is why that fund's available
  balance jumps from the June 30 baseline. There are no Expense rows yet,
  since that data isn't available on the business office side yet.

Because there's no spending recorded anywhere, every fund currently shows
as "Not spending down" or "Overspent" in the register. That's accurate, not
a bug: with zero expense entries logged so far, the dashboard has no
evidence any fund is being spent down, so it correctly declines to claim
otherwise. Once expense data starts coming in, whether through weekly
reconciliation exports or logged Expense rows, funds that are genuinely
being spent will start showing "On pace" again.

When you're ready to publish, you can either replace the June 30 row entirely
once you have two or more real exports to compare, or leave it as an
illustrative starting point while the real numbers accumulate around it.
Either way, the most recent row in the ledger is always the one people will
trust, and right now that's your real July 10 data.

## Password protection

The page now shows a password screen before the dashboard loads. It's worth
understanding exactly what this does and doesn't do, since this project
holds real fund data.

**What the built in screen actually is.** It's a light deterrent, not real
security. Everything that ships to a visitor's browser, including the
password check itself, can be read by opening the page's source or the
browser's developer tools. More importantly, `data/ledger.csv` is a plain
public file once this is on GitHub Pages. Anyone who knows or guesses that
address can fetch the fund numbers directly, whether or not they ever see
the password screen. Treat this the way you'd treat a "don't share this
link" request: fine for keeping the dashboard from turning up in casual
browsing or search engines, not something to rely on for data that truly
needs to stay confidential.

**Setting your own password.** The page ships with the password `changeme`
so you can see it work. Change it before you publish:
```
python3 -c "import hashlib; print(hashlib.sha256('yourpassword'.encode()).hexdigest())"
```
Copy the long string that prints out, then open `index.html`, find the line
with `PASSWORD_HASH`, and paste it in place of the existing value. The
plain password itself is never stored in the file, only this hash, which is
why generating it this way matters.

**If the data genuinely needs to stay private,** the honest answer is that
GitHub Pages can't do that on its own; a real login wall for a Pages site
requires GitHub Enterprise Cloud, which is aimed at large organizations, not
a single dashboard like this. A free option that actually works is to host
the exact same files on Netlify instead of GitHub Pages. This repo already
includes a `_headers` file with the settings Netlify needs: open it, replace
`dean:changeme` with your own username and password, then connect this
GitHub repository to a new Netlify site (netlify.com, "Add new site" →
"Import an existing project" → pick this repo, no build command needed,
publish directory is the repo root). Netlify will ask visitors for that
username and password before serving anything, including `data/ledger.csv`,
which is real, server enforced protection rather than a cosmetic screen.
This works on Netlify's free plan. You'd keep using GitHub for the data
updates exactly as described above; only where it's hosted changes.

## Notes on the pace/deadline calculation

For each fund, the dashboard compares its available balance at the first
export against the latest one to get a simple average weekly burn rate,
then projects forward to see whether the fund would hit zero before or
after May 31, 2027. This is a straight-line estimate, not a forecast - a
fund with only one or two exports so far will have a rough estimate that
firms up as more weekly data comes in. Funds that are currently overspent,
or that haven't shown any net spending yet, are flagged for review rather
than given a projected date.
