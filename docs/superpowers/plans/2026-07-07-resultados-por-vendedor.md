# Resultados por Vendedor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the empty space left by the removed "Desafio das Vendas" band with a 3-card "Resultados por Vendedor" section, fed by data already flowing through the existing realtime pipeline.

**Architecture:** `index.html` is a single static file (HTML+CSS+JS inline, no build step, deployed as-is to GitHub Pages). This feature adds CSS classes, an HTML section, and three DOM-render functions that read from the existing `state.records` global and the existing aggregation helpers (`aggregateData`, `getWeekDates`, `getMonthDates`, `sumTotals`). A new `renderResultsBand()` wrapper gets called from the existing `renderAll()`, which already runs on initial load and on every Supabase realtime event — so the new section is live-updating with zero new infrastructure.

**Tech Stack:** Vanilla JS, inline `<style>` CSS using existing `:root` variables, Supabase JS client v2 (already wired up). No npm, no bundler, no test framework in this repo — verification is manual (local static server) plus one isolated Node check for the one pure (non-DOM) helper function.

## Global Constraints

- Single file: all changes go into `index.html`. Do not introduce a build step, package.json, or test framework — none exists in this repo and the feature doesn't warrant one (YAGNI).
- Reuse existing CSS variables (`--diag`, `--plano`, `--venda`, `--pink`, `--bg-1`, `--border-1`, `--text-1/2/3/4`, `--font-display`, `--font-mono`) and existing data helpers (`aggregateData`, `getWeekDates`, `getMonthDates`, `sumTotals`, `formatCurrency`) — do not duplicate this logic.
- No new Supabase tables, no new columns, no new realtime channel. Only the existing `lancamentos_changes` channel and existing `renderAll()` pipeline are used.
- Visual placement: the new section goes at the very top of `<div class="container" id="mainContainer">`, immediately before the existing "Cadência Atual" section block.
- Local verification server: `python -m http.server 8080` from the repo root, then open `http://localhost:8080/index.html`. Only push to `main` (which auto-deploys to GitHub Pages) in the final task, after full manual QA.

---

### Task 1: HTML + CSS scaffold with a stubbed render hook

**Files:**
- Modify: `index.html` (CSS block ending at `</style>`, currently line 1217)
- Modify: `index.html` (HTML, immediately before the `<div class="section-meta">` that precedes `Cadência Atual`, currently line 1367)
- Modify: `index.html` (JS, inside `function renderAll()`, currently lines 2007-2015)

**Interfaces:**
- Produces: three empty DOM containers with ids `resultsMonthly`, `resultsWeekChart`, `resultsRevenue`, a meta-info element `resultsMeta`, and a JS function `renderResultsBand()` (stub in this task, filled in by Tasks 3-5) that later tasks will extend.

- [ ] **Step 1: Add the CSS block**

Open `index.html`. Find the line containing exactly `</style>` (currently line 1217, right after the `.hist-empty { ... }` rule). Insert the following block immediately before that `</style>` line:

```css
  /* RESULTADOS POR VENDEDOR */
  .results-grid {
    display: grid; grid-template-columns: repeat(3, 1fr);
    gap: 16px; margin-bottom: 48px;
  }
  .result-card {
    background: var(--bg-1); border: 1px solid var(--border-1);
    border-radius: 20px; padding: 24px;
  }
  .result-card-title {
    font-family: var(--font-display); font-size: 20px;
    font-weight: 400; letter-spacing: -0.02em;
    margin-bottom: 18px; color: var(--text-1);
  }
  .result-seller-row {
    display: flex; align-items: center; gap: 10px;
    padding: 10px 0; border-bottom: 1px solid var(--border-1);
  }
  .result-seller-row:last-child { border-bottom: none; }
  .result-seller-dot {
    width: 8px; height: 8px; border-radius: 50%;
    background: var(--seller-color); flex-shrink: 0;
  }
  .result-seller-name {
    font-family: var(--font-mono); font-size: 12px; font-weight: 700;
    color: var(--text-2); width: 48px; flex-shrink: 0;
  }
  .result-seller-stats {
    display: flex; gap: 14px; margin-left: auto;
    font-family: var(--font-mono); font-size: 12px; font-weight: 700;
  }
  .result-stat-diag { color: var(--diag); }
  .result-stat-plano { color: var(--plano); }
  .result-stat-venda { color: var(--venda); }
  .result-week-row {
    display: flex; align-items: center; gap: 10px; padding: 8px 0;
  }
  .result-week-name {
    font-family: var(--font-mono); font-size: 12px; font-weight: 700;
    color: var(--text-2); width: 40px; flex-shrink: 0;
  }
  .result-week-track {
    flex: 1; height: 22px; background: var(--bg-2);
    border-radius: 6px; overflow: hidden; border: 1px solid var(--border-1);
  }
  .result-week-fill {
    height: 100%; background: var(--seller-color); display: block;
    transition: width 1s cubic-bezier(0.16, 1, 0.3, 1);
  }
  .result-week-value {
    font-family: var(--font-mono); font-size: 12px; font-weight: 700;
    color: var(--text-1); width: 28px; text-align: right; flex-shrink: 0;
  }
  .result-revenue-row {
    display: flex; align-items: center; justify-content: space-between;
    padding: 10px 0; border-bottom: 1px solid var(--border-1);
  }
  .result-revenue-row:last-of-type { border-bottom: none; }
  .result-revenue-name {
    font-family: var(--font-mono); font-size: 12px; font-weight: 700;
    color: var(--text-2); display: flex; align-items: center; gap: 8px;
  }
  .result-revenue-value {
    font-family: var(--font-mono); font-size: 13px; font-weight: 700;
    color: var(--text-1);
  }
  .result-revenue-total {
    display: flex; align-items: center; justify-content: space-between;
    margin-top: 12px; padding-top: 14px; border-top: 2px solid var(--pink);
  }
  .result-revenue-total .result-revenue-name { font-size: 13px; color: var(--text-1); }
  .result-revenue-total .result-revenue-value {
    font-family: var(--font-display); font-size: 22px; color: var(--pink);
  }
```

- [ ] **Step 2: Add the HTML section**

Find this exact block (currently starting at line 1367):

```html
  <div class="section-meta">
    <div class="section-label">Cadência Atual</div>
    <div class="section-meta-info" id="cadenceMeta">—</div>
  </div>
  <div class="cadence-grid" id="cadenceGrid"></div>
```

Insert immediately **before** it (do not modify the block itself):

```html
  <div class="section-meta">
    <div class="section-label">Resultados por Vendedor</div>
    <div class="section-meta-info" id="resultsMeta">—</div>
  </div>
  <div class="results-grid">
    <div class="result-card">
      <div class="result-card-title">Mês por Vendedor</div>
      <div id="resultsMonthly"></div>
    </div>
    <div class="result-card">
      <div class="result-card-title">Comparativo da Semana</div>
      <div id="resultsWeekChart"></div>
    </div>
    <div class="result-card">
      <div class="result-card-title">Receita do Mês</div>
      <div id="resultsRevenue"></div>
    </div>
  </div>
```

- [ ] **Step 3: Add the stub `renderResultsBand()` function and wire it into `renderAll()`**

Find `function renderWeeklyComparison() {` in the `<script>` block. Immediately **before** it, insert:

```js
  function renderResultsBand() {
    const monthLabel = parseDate(state.currentDate).toLocaleDateString('pt-BR', { month: 'long', year: 'numeric' });
    document.getElementById('resultsMeta').textContent = `Atualizado · ${monthLabel}`;
  }

```

Then find `function renderAll() {` and its body (currently):

```js
  function renderAll() {
    renderCadenceCards();
    renderFunnel();
    renderSellers();
    renderRanking();
    renderWeeklyComparison();
    renderHistorical();
  }
```

Replace it with:

```js
  function renderAll() {
    renderCadenceCards();
    renderFunnel();
    renderSellers();
    renderRanking();
    renderWeeklyComparison();
    renderHistorical();
    renderResultsBand();
  }
```

- [ ] **Step 4: Verify in the browser**

Run: `python -m http.server 8080` from the repo root (`C:\Users\VENDE-C_USER\temp-debug\dashboard-vendas`), then open `http://localhost:8080/index.html` in a browser.

Expected: page loads with no console errors; a new "Resultados por Vendedor" heading with 3 empty-but-titled cards ("Mês por Vendedor", "Comparativo da Semana", "Receita do Mês") appears directly above "Cadência Atual"; the meta-info text next to the heading shows the current month/year (e.g. "Atualizado · julho de 2026").

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Add Resultados por Vendedor section scaffold (HTML+CSS, empty cards)"
```

---

### Task 2: Pure ranking helper for the weekly chart

**Files:**
- Modify: `index.html` (JS, add a new top-level function near the other pure helpers, e.g. right after `sumTotals`, currently around line 1494)
- Verification script (not committed): `<scratchpad>/rank-sellers-test.mjs`

**Interfaces:**
- Consumes: nothing from earlier tasks (pure function, takes plain data as arguments).
- Produces: `rankSellersByWeekTotal(sellers, weekAggregated)` — used by Task 4's `renderResultsWeekChart()`. Signature: `sellers` is an array of `{ id, name, color }` (same shape as the `SELLERS` constant); `weekAggregated` is a map `{ [sellerId]: { diagnosticos, planoAcao, vendas, valor } }` (same shape `aggregateData()` returns). Returns an array sorted descending by `total`, each item `{ id, name, color, total, pct }` where `pct` is 0-100 (100 = the highest-scoring seller).

- [ ] **Step 1: Write the failing test**

Create `C:\Users\VENDE-~1\AppData\Local\Temp\claude\C--Users-VENDE-C-USER\aca768b1-f859-4835-beda-e716129921b9\scratchpad\rank-sellers-test.mjs` with:

```js
import assert from 'node:assert/strict';

function rankSellersByWeekTotal(sellers, weekAggregated) {
  throw new Error('not implemented');
}

const sellers = [
  { id: 'gab', name: 'Gab', color: '#fecdd3' },
  { id: 'mau', name: 'Mau', color: '#e11d48' },
  { id: 'gi',  name: 'Gi',  color: '#9f1239' },
  { id: 'gui', name: 'Gui', color: '#f43f5e' },
];

// Test 1: sorts descending by total
const week1 = {
  gab: { diagnosticos: 1, planoAcao: 1, vendas: 0, valor: 0 },
  mau: { diagnosticos: 3, planoAcao: 2, vendas: 1, valor: 0 },
  gi:  { diagnosticos: 0, planoAcao: 0, vendas: 0, valor: 0 },
  gui: { diagnosticos: 2, planoAcao: 0, vendas: 0, valor: 0 },
};
const r1 = rankSellersByWeekTotal(sellers, week1);
assert.deepEqual(r1.map(r => r.id), ['mau', 'gui', 'gab', 'gi'], 'sort order');
assert.equal(r1[0].total, 6, 'mau total');
assert.equal(r1[0].pct, 100, 'top seller is 100%');
assert.equal(r1[2].pct, 33, 'gab pct rounded'); // 2/6 = 33.33 -> 33

// Test 2: all-zero week doesn't divide by zero
const week2 = {
  gab: { diagnosticos: 0, planoAcao: 0, vendas: 0, valor: 0 },
  mau: { diagnosticos: 0, planoAcao: 0, vendas: 0, valor: 0 },
  gi:  { diagnosticos: 0, planoAcao: 0, vendas: 0, valor: 0 },
  gui: { diagnosticos: 0, planoAcao: 0, vendas: 0, valor: 0 },
};
const r2 = rankSellersByWeekTotal(sellers, week2);
assert.equal(r2.every(r => r.pct === 0), true, 'all zero pct, no NaN/Infinity');

console.log('ALL PASS');
```

- [ ] **Step 2: Run it to verify it fails**

Run: `node "C:\Users\VENDE-~1\AppData\Local\Temp\claude\C--Users-VENDE-C-USER\aca768b1-f859-4835-beda-e716129921b9\scratchpad\rank-sellers-test.mjs"`
Expected: throws `Error: not implemented`

- [ ] **Step 3: Implement the real function in the test script and re-run until it passes**

Replace the stub `rankSellersByWeekTotal` at the top of the scratch script with:

```js
function rankSellersByWeekTotal(sellers, weekAggregated) {
  const ranked = sellers.map(s => {
    const d = weekAggregated[s.id] || { diagnosticos: 0, planoAcao: 0, vendas: 0 };
    const total = d.diagnosticos + d.planoAcao + d.vendas;
    return { id: s.id, name: s.name, color: s.color, total };
  }).sort((a, b) => b.total - a.total);
  const maxTotal = Math.max(...ranked.map(r => r.total), 1);
  return ranked.map(r => ({ ...r, pct: Math.round((r.total / maxTotal) * 100) }));
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `node "C:\Users\VENDE-~1\AppData\Local\Temp\claude\C--Users-VENDE-C-USER\aca768b1-f859-4835-beda-e716129921b9\scratchpad\rank-sellers-test.mjs"`
Expected: prints `ALL PASS` with no assertion errors.

- [ ] **Step 5: Copy the verified function into `index.html`**

In `index.html`, find `function sumTotals(data) {` and its closing `}` (currently lines 1490-1494). Immediately after that closing `}`, insert:

```js
  function rankSellersByWeekTotal(sellers, weekAggregated) {
    const ranked = sellers.map(s => {
      const d = weekAggregated[s.id] || { diagnosticos: 0, planoAcao: 0, vendas: 0 };
      const total = d.diagnosticos + d.planoAcao + d.vendas;
      return { id: s.id, name: s.name, color: s.color, total };
    }).sort((a, b) => b.total - a.total);
    const maxTotal = Math.max(...ranked.map(r => r.total), 1);
    return ranked.map(r => ({ ...r, pct: Math.round((r.total / maxTotal) * 100) }));
  }
```

- [ ] **Step 6: Reload the local server and confirm no console errors**

With `python -m http.server 8080` still running, reload `http://localhost:8080/index.html`.
Expected: no console errors (the function exists but isn't called by any render code yet, so the page looks identical to Task 1's result).

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "Add rankSellersByWeekTotal helper for weekly seller comparison"
```

---

### Task 3: Quadrado 1 — Mês por Vendedor

**Files:**
- Modify: `index.html` (JS, add `renderResultsMonthly()` and call it from `renderResultsBand()`)

**Interfaces:**
- Consumes: `SELLERS`, `aggregateData()`, `getMonthDates()` (existing globals); no dependency on Task 2's helper.
- Produces: `renderResultsMonthly()`, called from `renderResultsBand()`.

- [ ] **Step 1: Implement `renderResultsMonthly()`**

In `index.html`, inside `renderResultsBand()` (added in Task 1), add the function right above it:

```js
  function renderResultsMonthly() {
    const month = aggregateData(getMonthDates());
    document.getElementById('resultsMonthly').innerHTML = SELLERS.map(s => {
      const d = month[s.id];
      return `
        <div class="result-seller-row">
          <span class="result-seller-dot" style="--seller-color: ${s.color}"></span>
          <span class="result-seller-name">${s.name}</span>
          <span class="result-seller-stats">
            <span class="result-stat-diag">${d.diagnosticos}</span>
            <span class="result-stat-plano">${d.planoAcao}</span>
            <span class="result-stat-venda">${d.vendas}</span>
          </span>
        </div>`;
    }).join('');
  }

```

Then update `renderResultsBand()` to call it first:

```js
  function renderResultsBand() {
    renderResultsMonthly();
    const monthLabel = parseDate(state.currentDate).toLocaleDateString('pt-BR', { month: 'long', year: 'numeric' });
    document.getElementById('resultsMeta').textContent = `Atualizado · ${monthLabel}`;
  }
```

- [ ] **Step 2: Verify against the existing "Cadência Atual" cards (ground truth)**

With the local server running, reload the page as the "gestor" user (default). In the "Mês por Vendedor" card, add up each column across all 4 sellers:
- Sum of the diag column (pink numbers) across all 4 rows.
- Sum of the plano column across all 4 rows.
- Sum of the vendas column across all 4 rows.

Compare each sum against the "Mês" value shown in the corresponding "Cadência Atual" card above it (Diagnóstico card → Mês; Plano de Ação card → Mês; Vendas card → Mês).
Expected: the three sums match exactly (both come from the same `aggregateData(getMonthDates())` call, so this catches copy-paste/wiring mistakes, not calculation bugs).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Implement monthly per-seller breakdown card (Resultados por Vendedor)"
```

---

### Task 4: Quadrado 2 — Comparativo da Semana

**Files:**
- Modify: `index.html` (JS, add `renderResultsWeekChart()` and call it from `renderResultsBand()`)

**Interfaces:**
- Consumes: `rankSellersByWeekTotal(sellers, weekAggregated)` from Task 2 (exact signature above), `SELLERS`, `aggregateData()`, `getWeekDates()`.
- Produces: `renderResultsWeekChart()`, called from `renderResultsBand()`.

- [ ] **Step 1: Implement `renderResultsWeekChart()`**

In `index.html`, add this function right above `renderResultsBand()`:

```js
  function renderResultsWeekChart() {
    const week = aggregateData(getWeekDates());
    const ranked = rankSellersByWeekTotal(SELLERS, week);
    document.getElementById('resultsWeekChart').innerHTML = ranked.map(r => `
      <div class="result-week-row">
        <span class="result-week-name">${r.name}</span>
        <span class="result-week-track">
          <span class="result-week-fill" style="--seller-color: ${r.color}; width: ${r.pct}%"></span>
        </span>
        <span class="result-week-value">${r.total}</span>
      </div>
    `).join('');
  }

```

Update `renderResultsBand()`:

```js
  function renderResultsBand() {
    renderResultsMonthly();
    renderResultsWeekChart();
    const monthLabel = parseDate(state.currentDate).toLocaleDateString('pt-BR', { month: 'long', year: 'numeric' });
    document.getElementById('resultsMeta').textContent = `Atualizado · ${monthLabel}`;
  }
```

- [ ] **Step 2: Verify against the existing seller cards (ground truth)**

Reload the page. In "Comparativo da Semana", for each seller add up the bar's number to the seller's own week numbers shown in their "Performance Individual" card mini-funnel (Diag + Plano + Vendas, week values).
Expected: each seller's bar value equals `diag + plano + vendas` from their own card for the week; bars are ordered largest-to-smallest top-to-bottom; the largest bar visually fills the full width of its track (100%); a seller with zero activity this week shows an empty (0-width) bar, not a broken layout.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Implement weekly seller comparison chart (Resultados por Vendedor)"
```

---

### Task 5: Quadrado 3 — Receita do Mês

**Files:**
- Modify: `index.html` (JS, add `renderResultsRevenue()` and call it from `renderResultsBand()`)

**Interfaces:**
- Consumes: `SELLERS`, `aggregateData()`, `getMonthDates()`, `sumTotals()`, `formatCurrency()` (all existing).
- Produces: `renderResultsRevenue()`, called from `renderResultsBand()`.

- [ ] **Step 1: Implement `renderResultsRevenue()`**

In `index.html`, add this function right above `renderResultsBand()`:

```js
  function renderResultsRevenue() {
    const month = aggregateData(getMonthDates());
    const totalValor = sumTotals(month).valor;
    const rows = SELLERS.map(s => `
      <div class="result-revenue-row">
        <span class="result-revenue-name">
          <span class="result-seller-dot" style="--seller-color: ${s.color}"></span>${s.name}
        </span>
        <span class="result-revenue-value">${formatCurrency(month[s.id].valor * 1000)}</span>
      </div>
    `).join('');
    const totalRow = `
      <div class="result-revenue-total">
        <span class="result-revenue-name">Total do time</span>
        <span class="result-revenue-value">${formatCurrency(totalValor * 1000)}</span>
      </div>`;
    document.getElementById('resultsRevenue').innerHTML = rows + totalRow;
  }

```

Update `renderResultsBand()` to its final form:

```js
  function renderResultsBand() {
    renderResultsMonthly();
    renderResultsWeekChart();
    renderResultsRevenue();
    const monthLabel = parseDate(state.currentDate).toLocaleDateString('pt-BR', { month: 'long', year: 'numeric' });
    document.getElementById('resultsMeta').textContent = `Atualizado · ${monthLabel}`;
  }
```

- [ ] **Step 2: Verify against the existing seller cards and cadence card (ground truth)**

Reload the page. For each seller, compare their row in "Receita do Mês" against the "Receita do mês" value already shown at the bottom of their own "Performance Individual" card.
Expected: values match exactly (both come from `md.valor * 1000` / `aggregateData(getMonthDates())[s.id].valor * 1000`). Then compare "Total do time" against the "Mês" meta value shown under the "Vendas" Cadência Atual card (`valueMonth` there).
Expected: totals match exactly; the total row is visually distinct (bigger, pink, separated by the top border) from the per-seller rows above it.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Implement monthly revenue card with team total (Resultados por Vendedor)"
```

---

### Task 6: Realtime smoke test and deploy

**Files:**
- None (verification-only task, plus the `git push` that deploys everything from Tasks 1-5).

**Interfaces:**
- Consumes: the complete `renderResultsBand()` pipeline from Tasks 1-5, and the existing `handleRealtimeUpdate()` → `renderAll()` wiring (untouched by this feature).

- [ ] **Step 1: Confirm the section updates on realtime events**

With `python -m http.server 8080` running, open `http://localhost:8080/index.html` in two separate browser tabs (or one normal + one incognito window, so they get different `sessionId` values). In tab A, log in as a seller (e.g. "Gab") and change one of today's input values (e.g. Diagnóstico). In tab B (logged in as "gestor" or another seller), watch the "Resultados por Vendedor" section without reloading.
Expected: within a couple seconds, tab B's three cards update to reflect the new numbers, exactly as the rest of the dashboard (Cadência Atual, Performance Individual, etc.) already does — no manual refresh needed.

- [ ] **Step 2: Full visual pass**

Resize the browser window to check the 3-column grid doesn't overflow or break at typical desktop widths (1280px, 1440px, 1920px). Confirm the section sits directly above "Cadência Atual" with consistent spacing (`margin-bottom: 48px`, matching the other sections).

- [ ] **Step 3: Push to deploy**

```bash
git push
```

Expected: push succeeds; GitHub Pages rebuilds automatically (usually within ~1 minute) at https://mauriciocuencas-boop.github.io/dashboard-vendas/.

- [ ] **Step 4: Verify on the live site**

Open https://mauriciocuencas-boop.github.io/dashboard-vendas/ after the deploy finishes and confirm the "Resultados por Vendedor" section renders correctly with real production data (not just the local snapshot used during development).
