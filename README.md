# git-add-server.js-git-commit--m-Serve-index.html-at-git-push


tax-express-quebec/
  README.md
  package.json
  server.js
  .env.example
  constants/
    taxRates_2025_placeholders.js
  src/
    ui/
      index.html
      app.js
      styles.css


// server.js
// Tax Express Quebec (v1) - Hosted web app (A1)
// Uses placeholder 2025 constants.
// Later: replace constants/taxRates_2025_placeholders.js with 2026 numbers.

import express from "express";
import path from "path";
import { fileURLToPath } from "url";
import { computeTaxes } from "./constants/computeTaxes.js";
import { taxConstants } from "./constants/taxRates_2025_placeholders.js";

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

const app = express();
app.use(express.json());

const publicDir = path.join(__dirname, "src", "ui");
app.use(express.static(publicDir));

app.get("/api/health", (req, res) => res.json({ ok: true }));

app.post("/api/calc", (req, res) => {
  try {
    const input = req.body;
    const result = computeTaxes(input, taxConstants);
    res.json(result);
  } catch (e) {
    res.status(400).json({ error: e?.message || "Calculation error" });
  }
});

const port = process.env.PORT || 3000;
app.listen(port, () => {
  console.log(`Tax Express Quebec running on http://localhost:${port}`);
});


// constants/computeTaxes.js
// Computes federal + Quebec taxes using placeholder 2025-style structure.
// IMPORTANT: This is a scaffold: when you replace constants, results update.

export function computeTaxes(input, C) {
  // ----- Inputs (v1 option 1) -----
  // Editable totals come from the UI.
  const employmentIncome = num(input?.employmentIncomeTotal);
  const rrspDeduction = num(input?.rrspContributionTotal);

  const cppContrib = num(input?.cppContributionsTotal);
  const eiPremiums = num(input?.eiPremiumsTotal);

  // Withheld amounts (for reconciliation)
  const fedWithheld = num(input?.federalIncomeTaxWithheldTotal);
  const qcWithheld = num(input?.quebecIncomeTaxWithheldTotal);

  // Quebec-specific totals (from Quebec Relevé section)
  const qppContrib = num(input?.qppContributionsTotal);
  const qpippremiums = num(input?.qpipPremiumsTotal);

  // Minimal core credits toggles (v1)
  const basicPersonal = !!input?.basicPersonal;
  const maritalOrOtherFlags = input?.flags || {};

  // ----- Placeholders: Taxable income logic -----
  // Real Canada/Quebec systems have more layers; this is a simplified v1.
  const deductibleRRSP = Math.max(0, rrspDeduction);

  // Simplified taxable income base
  const federalTaxable = Math.max(0, employmentIncome - deductibleRRSP);
  const quebecTaxable = Math.max(0, employmentIncome - deductibleRRSP);

  // ----- Federal tax (placeholder brackets) -----
  const fed = calcProgressiveTax(federalTaxable, C.federal.brackets);

  // Non-refundable credit (placeholder)
  // In v1, we use only basic personal amount as a credit.
  let fedCredits = 0;
  if (basicPersonal) fedCredits += C.federal.basicCredit * C.federal.basicCreditRate;

  // Optional: could add CPP/EI refundable offsets if desired; v1 keeps it minimal.
  const fedNetTax = Math.max(0, fed - fedCredits);

  // ----- Quebec tax (placeholder brackets) -----
  const qc = calcProgressiveTax(quebecTaxable, C.quebec.brackets);

  let qcCredits = 0;
  if (basicPersonal) qcCredits += C.quebec.basicCredit * C.quebec.basicCreditRate;

  const qcNetTax = Math.max(0, qc - qcCredits);

  // ----- Reconciliation -----
  const federalBalance = fedNetTax - fedWithheld;
  const quebecBalance = qcNetTax - qcWithheld;

  // What to show in UI
  return {
    summary: {
      employmentIncome,
      rrspDeduction,
      taxable: {
        federal: federalTaxable,
        quebec: quebecTaxable,
      },
    },
    breakdown: {
      federal: {
        grossTax: fed,
        credits: fedCredits,
        netTax: fedNetTax,
        withheld: fedWithheld,
        balance: federalBalance,
      },
      quebec: {
        grossTax: qc,
        credits: qcCredits,
        netTax: qcNetTax,
        withheld: qcWithheld,
        balance: quebecBalance,
      },
      note:
        "Using 2025 placeholder constants (scaffold). Replace constants with 2026 official rates/thresholds to match exact results.",
    },
    usedInputs: {
      cppContrib,
      eiPremiums,
      qppContrib,
      qpippremiums,
      basicPersonal,
      maritalOrOtherFlags,
    },
  };
}

// ----- helpers -----
function num(v) {
  const n = Number(v);
  return Number.isFinite(n) ? n : 0;
}

function calcProgressiveTax(taxableIncome, brackets) {
  // brackets: [{ upTo: number|null, rate: number }]
  let remaining = taxableIncome;
  let tax = 0;
  let prevLimit = 0;

  for (const b of brackets) {
    const upTo = b.upTo;
    const rate = b.rate;

    if (upTo === null) {
      tax += remaining * rate;
      break;
    }

    const slice = Math.max(0, Math.min(remaining, upTo - prevLimit));
    tax += slice * rate;

    remaining -= slice;
    prevLimit = upTo;

    if (remaining <= 0) break;
  }
  return tax;
}

// constants/taxRates_2025_placeholders.js
// Replace these with official 2026 constants later.
// Structure is designed so you can swap values without changing code.

export const taxConstants = {
  federal: {
    // Placeholder brackets (rate/threshold structure)
    // upTo: upper limit of bracket; last bracket uses upTo: null
    brackets: [
      { upTo: 57375, rate: 0.15 },
      { upTo: 114750, rate: 0.205 },
      { upTo: 177, rate: 0.26 }, // placeholder intermediate (n/a)
      { upTo: null, rate: 0.29 },
    ],
    // Placeholder basic credit and credit rate
    basicCredit: 15200, // placeholder
    basicCreditRate: 0.15, // placeholder effective rate for non-refundable credits
  },

  quebec: {
    brackets: [
      { upTo: 51500, rate: 0.14 },
      { upTo: 103000, rate: 0.19 },
      { upTo: null, rate: 0.24 },
    ],
    basicCredit: 16600, // placeholder
    basicCreditRate: 0.149, // placeholder
  },
};

<!-- src/ui/index.html -->
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Tax Express Quebec</title>
  <link rel="stylesheet" href="styles.css" />
</head>
<body>
  <div class="container">
    <header class="hero">
      <h1>Tax Express Quebec</h1>
      <p>Federal + Québec v1 calculator with editable T4 &amp; Relevé-style totals (2025 placeholders for now).</p>
    </header>

    <form id="form">
      <section class="card">
        <h2>Client basics (v1)</h2>
        <label class="row">
          <span>Province</span>
          <select id="province" disabled>
            <option selected>Québec</option>
          </select>
        </label>

        <label class="row">
          <span>Basic personal amount</span>
          <input type="checkbox" id="basicPersonal" checked />
        </label>
      </section>

      <section class="card">
        <h2>T4 Summary (editable totals)</h2>
        <div class="grid">
          <label>
            <span>Employment income total</span>
            <input type="number" id="employmentIncomeTotal" step="0.01" value="0" />
          </label>

          <label>
            <span>CPP contributions total (federal)</span>
            <input type="number" id="cppContributionsTotal" step="0.01" value="0" />
          </label>

          <label>
            <span>EI premiums total</span>
            <input type="number" id="eiPremiumsTotal" step="0.01" value="0" />
          </label>

          <label>
            <span>Federal income tax withheld (optional)</span>
            <input type="number" id="federalIncomeTaxWithheldTotal" step="0.01" value="0" />
          </label>

          <label>
            <span>RRSP contribution total (deduction)</span>
            <input type="number" id="rrspContributionTotal" step="0.01" value="0" />
          </label>
        </div>
        <p class="hint">For v1, only key fields are used in the placeholder calculation engine.</p>
      </section>

      <section class="card">
        <h2>Québec Relevé (editable totals)</h2>
        <div class="grid">
          <label>
            <span>QPP contributions total</span>
            <input type="number" id="qppContributionsTotal" step="0.01" value="0" />
          </label>

          <label>
            <span>QPIP premiums total (optional)</span>
            <input type="number" id="qpipPremiumsTotal" step="0.01" value="0" />
          </label>

          <label>
            <span>Québec income tax withheld (optional)</span>
            <input type="number" id="quebecIncomeTaxWithheldTotal" step="0.01" value="0" />
          </label>
        </div>
      </section>

      <div class="actions">
        <button type="button" id="btnCalc">Calculate</button>
      </div>
    </form>

    <section class="card" id="results" style="display:none;">
      <h2>Results</h2>

      <div class="summary">
        <div class="kpi">
          <div class="label">Federal net tax</div>
          <div class="value" id="fedNetTax">$0</div>
        </div>
        <div class="kpi">
          <div class="label">Québec net tax</div>
          <div class="value" id="qcNetTax">$0</div>
        </div>
      </div>

      <div class="breakdown">
        <h3>Federal</h3>
        <div class="row">
          <span>Gross tax</span><strong id="fedGross">$0</strong>
        </div>
        <div class="row">
          <span>Credits</span><strong id="fedCredits">$0</strong>
        </div>
        <div class="row">
          <span>Withheld</span><strong id="fedWithheld">$0</strong>
        </div>
        <div class="row">
          <span>Balance (refund / owing)</span><strong id="fedBal">$0</strong>
        </div>

        <hr />

        <h3>Québec</h3>
        <div class="row">
          <span>Gross tax</span><strong id="qcGross">$0</strong>
        </div>
        <div class="row">


// src/ui/app.js
const $ = (id) => document.getElementById(id);

function money(n) {
  const x = Number(n);
  if (!Number.isFinite(x)) return "$0";
  return x.toLocaleString("en-CA", { style: "currency", currency: "CAD" });
}

function readForm() {
  return {
    basicPersonal: $("basicPersonal").checked,

    employmentIncomeTotal: $("employmentIncomeTotal").value,
    cppContributionsTotal: $("cppContributionsTotal").value,
    eiPremiumsTotal: $("eiPremiumsTotal").value,
    rrspContributionTotal: $("rrspContributionTotal").value,

    federalIncomeTaxWithheldTotal: $("federalIncomeTaxWithheldTotal").value,

    qppContributionsTotal: $("qppContributionsTotal").value,
    qpipPremiumsTotal: $("qpipPremiumsTotal").value,
    quebecIncomeTaxWithheldTotal: $("quebecIncomeTaxWithheldTotal").value,

    flags: {},
  };
}

$("btnCalc").addEventListener("click", async () => {
  const input = readForm();
  const res = await fetch("/api/calc", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(input),
  });

  const data = await res.json();
  if (!res.ok) {
    alert(data?.error || "Calculation failed");
    return;
  }

  // Show results
  $("results").style.display = "block";

  const fed = data.breakdown.federal;
  const qc = data.breakdown.quebec;

  $("fedNetTax").textContent = money(fed.netTax);
  $("qcNetTax").textContent = money(qc.netTax);

  $("fedGross").textContent = money(fed.grossTax);
  $("fedCredits").textContent = money(fed.credits);
  $("fedWithheld").textContent = money(fed.withheld);
  $("fedBal").textContent = money(fed.balance);

  $("qcGross").textContent = money(qc.grossTax);
  $("qcCredits").textContent = money(qc.credits);
  $("qcWithheld").textContent = money(qc.withheld);
  $("qcBal").textContent = money(qc.balance);

  $("note").textContent = data.breakdown.note;
});

/* src/ui/styles.css */
:root { --bg:#0b1220; --card:#111b33; --text:#e8eefc; --muted:#a9b6d6; --accent:#6aa8ff; }
* { box-sizing: border-box; }
body { margin:0; font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial; background: var(--bg); color: var(--text); }
.container { max-width: 980px; margin: 24px auto; padding: 0 16px; }
.hero { margin-bottom: 16px; }
.hero h1 { margin:0 0 8px 0; }
.hero p { margin:0; color: var(--muted); }

.card { background: var(--card); border: 1px solid rgba(255,255,255,0.08); border-radius: 14px; padding: 16px; margin-bottom: 14px; }
.card h2 { margin:0 0 12px 0; font-size: 18px; }
h3 { margin: 14px 0 8px 0; }

.row { display:flex; align-items:center; justify-content: space-between; gap: 12px; margin: 10px 0; }
.grid { display:grid; gap: 12px; grid-template-columns: repeat(2, minmax(0, 1fr)); }
@media (max-width: 720px) { .grid { grid-template-columns: 1fr; } }

label { display:flex; flex-direction: column; gap: 6px; }
label span { color: var(--muted); font-size: 13px; }

input, select {
  background: rgba(255,255,255,0.04);
  border: 1px solid rgba(255,255,255,0.12);
  color: var(--text);
  border-radius: 10px;
  padding: 10px;
  outline: none;
}

.actions { display:flex; gap: 12px; margin: 10px 0 18px; }
button {
  background: var(--accent);
  color: #061023;
  border: none;
  font-weight: 700;
  padding: 12px 18px;
  border-radius: 12px;
  cursor: pointer;
}
button:hover { filter: brightness(1.05); }

.hint { margin: 10px 0 0; color: var(--muted); font-size: 13px; }

.summary { display:flex; gap: 12px; margin-bottom: 10px; }
.kpi { flex:1; background: rgba(255,255,255,0.04); border: 1px solid rgba(255,255,255,0.08); border-radius: 12px; padding: 12px; }
.kpi .label { color: var(--muted); font-size: 13px; margin-bottom: 6px; }
.kpi .value { font-size: 22px; font-weight: 800; }

.breakdown .row { margin: 6px 0; }
hr { border:0; border-top: 1px solid rgba(255,255,255,0.1); margin: 14px 0; }
.note { color: var(--muted); font-size: 13px; }

{
  "name": "tax-express-quebec",
  "version": "1.0.0",
  "type": "module",
  "private": true,
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.19.2"
  }
}


cd tax-express-quebec
npm install
node server.js
# open http://localhost:3000


          <span>Credits</span><strong id="qcCredits">$0</strong>
        </div>
        <div class="row">
          <span>Withheld</span><strong id="qcWithheld">$0</strong>
        </div>
        <div class="row">
          <span>Balance (refund / owing)</span><strong id="qcBal">$0</strong>
        </div>
      </div>

      <p class="note" id="note"></p>
    </section>
  </div>

  <script type="module" src="app.js"></script>
</body>
</html>


