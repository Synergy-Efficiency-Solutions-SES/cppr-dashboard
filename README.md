# CPPR Energy Dashboard

Internal tool for Synergy Efficiency Solutions (SES) to turn monthly chiller-plant Excel
exports (CaaS billing data) into an interactive performance dashboard, and to publish a
simplified read-only report for each client via a stable public link.

This repo contains two kinds of files:

| File | What it is | Who sees it |
|---|---|---|
| `index.html` | The full internal tool: upload Excel files, browse all 11 dashboard pages, export client reports | SES team only |
| `<client>-<token>.html` (e.g. `mustika-57871761.html`) | A static, read-only report for one client (Monthly + Yearly Overview only) | The specific client it was made for |
| `robots.txt` | Tells search engines not to index anything in this repo | N/A |

The whole thing is a single-page app — no backend, no database, no build step. All Excel
parsing happens client-side in the browser using [SheetJS/xlsx.js](https://github.com/SheetJS/sheetjs)
and charts are drawn with [Chart.js](https://www.chartjs.org/), both loaded from a public CDN.

---

## 1. Monthly workflow (what you actually do every month)

1. Open the live tool: `https://synergy-efficiency-solutions-ses.github.io/cppr-dashboard/`
   (or open `index.html` locally in a browser — works the same way, no server needed).
2. Drag & drop the client's Excel file(s) or folder onto the drop zone. The tool parses them
   entirely in your browser — nothing is uploaded anywhere.
3. Browse the dashboard (Monthly, Yearly, Chiller Plant, Trending, etc.) to check the data
   looks right.
4. Click **"⬇ Export Client View"**. This downloads a single self-contained `.html` file
   named `<client>-<token>.html` (see §3 for what the token is and why it's stable).
5. Upload that downloaded file to this GitHub repo (same filename — don't rename it), replacing
   the previous month's version for that client.
6. Push / commit. Wait ~1-2 minutes for GitHub Pages to redeploy.
7. The client's link (same URL every month — see §3) now shows the updated data. You don't
   need to re-send them anything.

**First time exporting a given client**: note down the resulting URL somewhere (a spreadsheet,
a notes file — anything outside this repo) so you don't have to re-derive it later. It will
never change for that client unless you rename them in your source data (see §3).

---

## 2. How to find a client's public URL

```
https://synergy-efficiency-solutions-ses.github.io/cppr-dashboard/<filename>
```

The `<filename>` is exactly what the browser downloaded when you clicked Export (check your
Downloads folder, or the browser's download popup). It looks like:

```
mustika-57871761.html
```

---

## 3. How the per-client filename/token works

Each client gets a filename made of two parts: `<slugified-client-name>-<token>.html`.

The **token** is an 8-character hex string computed deterministically from the client's name,
so the **same client name always produces the same token**, forever, on any computer:

```javascript
const EXPORT_URL_SALT = 'ses-cppr-2026-change-me-if-needed';

function clientExportToken(clientName) {
  const str = EXPORT_URL_SALT + '::' + clientName.toLowerCase().trim();
  let h = 2166136261; // FNV-1a hash, offset basis
  for (let i = 0; i < str.length; i++) {
    h ^= str.charCodeAt(i);
    h = Math.imul(h, 16777619); // FNV prime
  }
  return (h >>> 0).toString(16).padStart(8, '0');
}
```

This is the [FNV-1a hash](https://en.wikipedia.org/wiki/Fowler%E2%80%93Noll%E2%80%93Vo_hash_function)
combined with a fixed salt string. Both `index.html` (this tool) and every exported client
file use the exact same function, defined in two places in the code (see §6).

### Why this exists

This repo is a **public** GitHub repo (GitHub Pages requires a paid plan for private repos +
Pages). Client reports contain business data (energy consumption, costs, savings) that
shouldn't be casually discoverable. A predictable filename like `mustika.html` could be
guessed or crawled by search engines. This token scheme, combined with `robots.txt` (§4),
makes that impractical without actually knowing the link.

### What this does and doesn't protect against

- ✅ Stops search engines from indexing/surfacing client reports
- ✅ Stops casual URL-guessing (`client1.html`, `client2.html`, etc.)
- ❌ **Not cryptographic security.** Anyone who reads this tool's source code (`index.html`,
  itself public) AND knows a client's exact name can recompute their token. This is
  considered an acceptable trade-off because only the SES team realistically looks at this
  tool's source, and the goal is to stop opportunistic discovery, not a targeted attacker.
- ❌ Does not hide the report from someone who already has the direct link (by design — the
  client needs to be able to open it).

### ⚠️ If you ever change `EXPORT_URL_SALT`

**Every existing client link will break** (new token = new filename = old URL 404s). Don't
change it unless you're prepared to re-export and re-send every client a new link. There is
no reason to change it under normal use.

### Keeping the same client name across months

The token depends on the *exact* client name string used in your Excel data / folder names.
`Mustika`, `mustika`, and `Mustika Hotel` all produce **different** tokens. Keep the client
name spelled identically every month, or the export will silently generate a new filename
and the client's existing link will stop working.

---

## 4. `robots.txt`

Sits at the repo root, applies to the whole site:

```
User-agent: *
Disallow: /
```

This tells all search engine crawlers not to index anything here. `index.html` (the internal
tool) has no client data in it, so blocking it costs nothing — it's just not meant to be
found via Google either. If you ever add a page to this repo that *should* be publicly
indexable, this blanket rule would need to be narrowed (ask before changing it if unsure).

---

## 5. Architecture overview (for anyone modifying the code)

Everything lives in a small number of large files:

- **`index.html`** — the whole internal tool. Structure, in order:
  - `<style>` — all CSS (design tokens as CSS variables at the top: `--ses-blue`, `--ses-green`, etc.)
  - HTML for the drop zone, header/nav, and each of the 11 dashboard pages (`<div class="page" id="page-...">`)
  - `<script>` — all JavaScript:
    - **Parsing**: `parseFile()` reads an uploaded `.xlsx` (via SheetJS) and extracts each
      relevant sheet (Billing Data, Weekly Breakdown, per-chiller part-load sheets, Raw Data,
      Comments, etc.) into a plain JS object.
    - **Pure computation functions** (no DOM access): `computeMonthlyStats(dataSubset)` and
      `computeYearlyStats(dataSubset)` — take filtered data in, return all the numbers a page
      needs. Keeping these separate from rendering means the same formulas can be reused by
      both the live dashboard *and* the exported client file (see below) without duplicating
      the math.
    - **Render functions**: `renderMonthly()`, `renderYearly()`, `renderDetailed()`, etc. —
      call the compute functions above, then push results into the DOM and Chart.js.
    - **`exportClientView()`** — builds the standalone client HTML file as a big JS template
      string, and triggers the browser download.

- **Exported client files** (`<client>-<token>.html`) — generated entirely by
  `exportClientView()`. Not meant to be hand-edited; regenerate from `index.html` instead.

### ⚠️ The most important thing to know before editing

**The exported client file is a near-complete second copy of the dashboard**, embedded as a
JS string inside `exportClientView()` in `index.html`. It has its own copies of:
`computeMonthlyStats`, `computeYearlyStats`, `renderMonthly`, `renderYearly`, `clientExportToken`,
and various formatting helpers (`fmt4`, `fmtNum`, `periodLabel`, `chartOpts`, etc.).

**If you change a formula, a KPI calculation, or a chart in the main dashboard's
`computeMonthlyStats`/`computeYearlyStats`/`renderMonthly`/`renderYearly`, you must make the
identical change inside the `exportClientView()` template string too** — search for the
comment `// ── Shared pure computation` inside that function to find the embedded copies.
Nothing enforces these staying in sync automatically; it's on you (or whoever edits this) to
remember. This is the single biggest source of "the client report doesn't match the live
dashboard" bugs if missed.

---

## 6. How to modify or upgrade the code

1. **Edit `index.html` directly** — it's plain HTML/CSS/JS, no build step, no dependencies
   to install. Open it in any text editor.
2. **If you touch anything inside `exportClientView()`'s template string**, remember it's
   JavaScript-inside-a-JavaScript-string: the inner `<script>` tag is escaped as `<\/script>`
   so it doesn't prematurely close the outer one. Don't "fix" that escaping — it's required.
3. **Check your syntax before deploying.** With [Node.js](https://nodejs.org) installed, you
   can sanity-check the script blocks without opening a browser:
   ```bash
   node --check path/to/extracted-script.js
   ```
   (You'll need to manually extract the `<script>...</script>` contents into a `.js` file
   first — there's no automated build/test pipeline in this repo.)
4. **Test locally** by just opening `index.html` in a browser before pushing — no server
   required, drag-and-drop and export both work fully offline against local files.
5. **Push to `main`.** GitHub Pages redeploys automatically, usually within 1-2 minutes.
6. **Hard-refresh** (Ctrl+Shift+R / Cmd+Shift+R) when checking the live site — browsers
   aggressively cache static HTML, and a normal refresh can show you a stale cached copy.
7. **Check the footer** on the Home page: it shows "Last modified: [date]", read
   automatically from the deployed file's metadata (`document.lastModified`). If that date
   doesn't match your latest push, you're looking at a cached copy — hard-refresh again.
   *(Known limitation: this depends on GitHub Pages' CDN reporting an accurate
   Last-Modified header. It has worked reliably in testing so far, but if it ever starts
   showing an obviously wrong or frozen date, that's this limitation, not a dashboard bug.)*

---

## 7. Known limitations / things to keep in mind

- **Single public repo, no access control.** Anyone with a client's exact link can view
  their report. See §3 for the mitigations in place and their limits.
- **No automated tests.** Changes are verified manually (open in browser, check the pages,
  `node --check` for syntax). Consider this before making large refactors.
- **Excel column parsing is position-based in places** (e.g. `row[1]`, `row[8]`) rather than
  header-based, for some sheets. If someone inserts/reorders a column in the source Excel
  template, parsing can silently produce wrong numbers instead of an error. Worth checking
  after any change to the Excel template SES uses upstream.
- **Exported file size grows with the amount of data embedded.** Minute-level `rawData` is
  deliberately stripped from client exports to keep file size reasonable; if you export many
  months of billing/weekly data for one client, the file can still get large — no hard limit
  is enforced.
- **Chart.js and SheetJS are loaded from a public CDN** (`cdnjs.cloudflare.com`,
  `jsdelivr.net`). This is fine as long as the deployed dashboard is only ever opened with
  internet access (true for both the internal tool via GitHub Pages and client reports opened
  as a web link). If this tool is ever used fully offline, charts would fail to render.

---

## 8. Quick troubleshooting

| Symptom | Likely cause |
|---|---|
| Downloaded export filename has no `-xxxxxxxx` suffix | You're using a cached/old version of `index.html`. Hard-refresh and re-export. |
| Client says their link doesn't work anymore | Check the client name is spelled identically to previous months — a changed name produces a new token/URL (§3). |
| Live site doesn't reflect your latest push | Hard-refresh (Ctrl+Shift+R). Check the footer's "Last modified" date. |
| A KPI differs between the live dashboard and a client's exported report | The two `computeMonthlyStats`/`computeYearlyStats` copies have drifted out of sync — see §5. |
| Empty gray boxes in a KPI grid | Should be fixed as of the Flexbox-based `.kpi-grid` layout — if it recurs, check that fix wasn't accidentally reverted. |
