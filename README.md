# IHPS Fluid Calculator

> Clinical decision-support tool for fluid and electrolyte management in infants with **Infantile Hypertrophic Pyloric Stenosis (IHPS)** — deterministic protocol logic, deployed to production on Render. **No patient data is included in this repository.**

IHPS infants typically arrive with a **hypochloremic, hypokalemic metabolic alkalosis** from persistent vomiting. Pyloromyotomy is unsafe until chloride, potassium, and acid–base status are corrected. That pre-op resuscitation decision is repetitive, parameter-heavy, and easy to get subtly wrong under time pressure.

**The engineering job:** turn admission labs + weight into a repeatable fluid plan — fluid type, 1.5× maintenance rate, bolus, correction window, recheck interval, and alerts — keyed to severity thresholds from retrospective protocol calibration (private, not published). It supports a specific pre-op workflow, not generic pediatrics. The logic is deterministic and auditable: every output traces to a named rule, same inputs always produce the same plan, no model call in the loop.

---

## What it does

A clinician submits labs + weight to `POST /calculate` and gets back a JSON plan:

- **Fluid selection** — chosen from sodium and (if provided) glucose
- **Infusion rate** — 1.5× maintenance (6 mL/kg/hr for the first 10 kg)
- **Potassium replacement** — KCl concentration scaled to hypokalemia severity
- **Bolus recommendation** — 10–20 mL/kg NS by depletion severity
- **Estimated correction time** — 12–36 hours, by severity
- **Lab recheck interval** — 12–24 hours, by risk
- **Clinical alerts** — critical/warning flags surfaced before any action

---

## How it works

### Inputs

**Essential (required):** `Sodium` · `Potassium` · `Chloride` · `pH` · `Weight`

**Optional (refine severity, bolus boosters, and alerts):** `Glucose` · `Creatinine` · `BUN` · `Hematocrit` · `Lactate` · `pCO₂` · `Base Excess` · `HCO₃`

**Input validation** (rejects out-of-range form data): Na 120–155 · K 2.0–8.0 · Cl 60–120 · pH 7.0–7.7 · weight 1.0–10.0 kg.

### Fluid selection

The live path (`selectOptimalFluidEnhanced`, wired into `calculateIHPSFluidPlan`) uses **sodium with a 135 mmol/L gate**, modified by glucose when entered:

| Glucose (mmol/L) | Sodium (mmol/L) | Fluid |
|---|---|---|
| < 2.5 | < 135 | D10 ½ NS + KCl |
| < 2.5 | ≥ 135 | D10 NS + KCl |
| 2.5 – < 3.0 | < 135 | D10 ½ NS + KCl |
| 2.5 – < 3.0 | ≥ 135 | D5 ½ NS + KCl |
| ≥ 3.0 *or omitted* | < 135 | NS + 5% Dextrose + KCl |
| ≥ 3.0 *or omitted* | ≥ 135 | D5 ½ NS + KCl |

**Potassium modifiers** (applied after fluid choice):

| Potassium (mmol/L) | KCl in bag |
|---|---|
| < 3.0 | 40 mEq/L |
| > 5.0 | KCl removed |
| otherwise | 20 mEq/L |

### Bolus / resuscitation

| Severity | Triggers | Bolus |
|---|---|---|
| Severe | Na < 130 **or** Cl < 70 **or** pH > 7.55 | 20 mL/kg NS |
| Moderate | Na < 135 **or** Cl < 90 **or** pH > 7.45 | 10 mL/kg NS |
| Booster | Hct > 50 **or** lactate > 2.0 | at least 10 mL/kg |

### Clinical alerts

The API returns string alerts (`generateSimpleAlerts`):

| Level | Condition | Theme |
|---|---|---|
| CRITICAL | K < 3.0 | Severe hypokalaemia → aggressive KCl |
| CRITICAL | K > 5.5 | Hyperkalaemia risk → hold KCl |
| WARNING | Na < 130 | Severe hyponatraemia → NS + 5% dextrose |
| WARNING | Cl < 70 | Severe hypochloraemia → longer correction |
| WARNING | pH > 7.55 | Severe alkalosis → slow correction |

### Correction time & recheck

- **Estimated correction time:** 12 / 24 / 36 hours, banded by admission severity
- **Lab recheck interval:** 12–24 hours, by risk

---

## API

Single endpoint. Browser sends JSON, server returns the plan as JSON.

```http
POST /calculate
Content-Type: application/json

{
  "sodium": 132,
  "potassium": 2.8,
  "chloride": 88,
  "ph": 7.48,
  "weight": 4.2,
  "glucose": 3.1
}
```

```json
{
  "success": true,
  "fluidType": "D5 1/2 NS + 20 mEq/L KCl",
  "fluidSelectionReason": "...",
  "fluidRatePerHour": 25,
  "maintenanceRate": 17,
  "correctionTime": 24,
  "totalFluidVolume": 600,
  "bolusRecommendation": {
    "volume": 10,
    "totalVolume": 42,
    "reason": "..."
  },
  "alerts": [],
  "recheckInterval": "12 hours",
  "nextLabCheck": "Next lab check in 12 hours (critical first 24 hours)"
}
```

### Synthetic examples

The [`examples/`](examples/) directory contains **fictional** lab payloads for demos and `curl` tests — not real patients. See [examples/README.md](examples/README.md).

---

## Architecture

| Layer | Implementation |
|---|---|
| Runtime | Node.js ≥ 20 (`render.yaml` pins 20.11.0) |
| Backend | Express 4 — single `server.js`, one route: `POST /calculate` |
| Frontend | Vanilla HTML/CSS/JS in `public/index.html` — no framework, no build step |
| Data flow | Browser `fetch('/calculate')` with JSON body → server returns JSON plan |
| Core logic | Deterministic rules engine (`calculateIHPSFluidPlan` → `selectOptimalFluidEnhanced`); pure functions, no external model calls |
| Hosting | Render (`render.yaml`: `npm install` → `npm start`) |

A deterministic design was deliberate: in a clinical setting every recommendation must be explainable and reproducible. Same inputs, same plan, each output traceable to a rule.

---

## Running locally

```bash
git clone https://github.com/<you>/ihps-fluid-calculator.git
cd ihps-fluid-calculator
npm install
npm start
```

Server runs on `PORT` or defaults to **http://localhost:3000**. Open **http://localhost:3000** for the UI.

Try a synthetic case:

```bash
curl -s -X POST http://localhost:3000/calculate \
  -H "Content-Type: application/json" \
  -d @examples/moderate-depletion.request.json
```

---

## Deployment

Deployed to production on **Render** via `render.yaml`:

- Service: `ihps-fluid-calculator`
- Build command: `npm install`
- Start command: `npm start`

Expected public URL pattern (the repo doesn't store it — **confirm in your Render dashboard**):

```
https://ihps-fluid-calculator.onrender.com
```

---

## Validation & privacy

**What this repo contains:** application source code and **synthetic** example payloads only.

**What it does not contain:** real patient records, identifiers, registration numbers, exports, or clinical analysis documents. Thresholds in `server.js` were calibrated retrospectively against a **private** IHPS cohort that stays off GitHub.

**Clinical scope:** decision-support encoding of a unit-style pre-op fluid protocol — **not** a prospective trial, regulatory clearance, or peer-reviewed validation. Every output requires clinician review.

**For contributors:** do not commit `.csv`, `.xlsx`, `.pdf`, or other files that could hold PHI. `.gitignore` blocks common export types.

---

## Safety, scope & limitations

- **Decision support only.** Does not make treatment decisions or replace clinical judgment.
- **No patient data in repo.** Demo and test values are invented; do not paste real charts into issues or PRs.
- **Not a medical device.** No regulatory clearance. Not for unsupervised clinical use.
- **Narrow scope.** IHPS pre-op fluid management only. Not validated for other causes of pediatric dehydration or electrolyte disturbance.

---

## Roadmap

- Automated tests against `examples/*.request.json` (protocol regression, not PHI)
- Surface the decision trace alongside each recommendation (show *why*)
- Input/output audit log for clinical governance

---

## License

MIT — see [LICENSE](LICENSE).

---

## Author

**Bisma Majeed** — Forward Deployed Engineer
[LinkedIn](https://linkedin.com/in/bismaajeed)

---

## Disclaimer

This calculator provides clinical guidance only and should not replace clinical judgment. All recommendations should be reviewed by qualified medical professionals before implementation.
