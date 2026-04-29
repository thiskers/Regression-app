# Regression-app

A single-file browser tool for **statistical regression analysis** — load a sample or your own CSV, fit OLS / polynomial / logistic models, inspect a full diagnostic suite, and read a plain-English interpretation of what the model says.

No backend. No build step. Open `index.html` and start.

---

## Run

Just open `index.html` in any modern browser. Libraries are loaded from jsDelivr CDN; once cached, the file works offline.

```sh
# from the project root
xdg-open index.html      # Linux
open index.html          # macOS
start index.html         # Windows
```

Or host it from a tiny static server:

```sh
python3 -m http.server 8000
# then visit http://localhost:8000/
```

---

## What it does

- **8 bundled sample datasets** — Anscombe's quartet, mtcars, Longley (NIST), King County housing, NYC airquality, R cars, iris (binary), Pima diabetes
- **CSV upload + paste** with auto-typed columns, missing-data policies (listwise / mean / median impute), categorical dummy encoding, transforms (log / sqrt / square / standardise), polynomial expansion (degree 2–6)
- **OLS fit via QR** with full classical inference: SE, t / p / 95% CI / standardised β, R² / adj R², AIC / BIC / log-likelihood, overall F-test, HC3 robust SEs (toggle)
- **Logistic regression via IRLS** with Wald z-tests, McFadden pseudo-R², LR χ², AUC, calibration deciles, confusion matrix, separation detection
- **Full diagnostic suite**: leverage, Cook's distance, VIF, Durbin-Watson, Breusch-Pagan, Jarque-Bera, Shapiro-Wilk, plus all 13 standard diagnostic plots (Plotly: scatter+fit with CI/PI, residuals-vs-fitted with LOWESS, Q-Q, scale-location, residuals-vs-leverage with Cook's contours, residual histogram with N(0,σ̂) overlay, predictor correlation heatmap, partial-regression panels, predicted-vs-actual, coefficient forest, influence index, ROC, calibration)
- **Plain-English interpretation engine** — 15 rules across Overall fit / Predictor effects / Assumption issues / Recommendations, with a one-line headline that names the dominant story (high R² + heteroscedasticity → "interpret coefficients with caution"; perfect leverage point → "row 8 is influential"; etc.)
- **Predictions tab** — single-row form with auto-filled means, gives ŷ + 95% CI + 95% PI for OLS, p̂ + 95% CI + class for logistic
- **Export** — JSON state dump (full payload, matrices serialised), Markdown report (interpretation + coefficient table + diagnostics), or PDF via `window.print()` with a print-friendly stylesheet
- **In-app guided tutorial** — 11-step OLS tour + 4-step logistic branch with spotlight overlay (Esc to exit, ←/→ to navigate)

---

## Numerical accuracy

Validated against the [NIST StRD](https://www.itl.nist.gov/div898/strd/) reference datasets and R `lm()` canonical values. Open the **Self-test** page (footer link or `index.html#selftest`) to run the harness in your browser.

| Suite                                             | Tests | Status |
|---------------------------------------------------|------:|--------|
| NIST Norris (well-conditioned linear, n=36)       |     6 | PASS   |
| NIST Longley (cond ≈ 5×10⁹, n=16)                 |     8 | PASS   |
| NIST Wampler1 (perfect-fit polynomial degree 5)   |     8 | PASS   |
| R `lm(mpg ~ wt + hp + cyl, mtcars)` canonicals    |    10 | PASS   |
| Distribution functions (vs R 4.x reference)       |    20 | PASS   |
| **Total**                                         | **52**| **PASS**|

Tolerances:

- Norris (well-conditioned): β / SE / RSS to **1e-10** relative
- Longley (severely ill-conditioned): β / RSS to **1e-6** relative — well past NIST's 6-digit pass threshold
- Wampler1 (perfect fit): every β to 1e-8, RSS = 0 to 1e-10
- Distribution functions: 1e-7 to 1e-12 depending on algorithm (Acklam normInv, A&S 7.1.26 erf, Lentz continued fractions for incomplete beta / gamma)

The QR-based OLS solver (via `ml-matrix`) is what makes the ill-conditioned Longley test pass — naive normal-equations solvers lose 8+ digits on this problem.

---

## Stack

Single HTML file, no build step. Libraries loaded from jsDelivr (pinned versions):

| Library | Version | Used for |
|---------|---------|----------|
| Pico CSS    | 2     | semantic styling base |
| ml-matrix   | 6.10.7| QR decomposition + matrix ops for OLS / inverse for IRLS Hessian |
| Plotly.js   | 2.35.2| 13 diagnostic plots, dark-theme aware, native PNG export |
| Papa Parse  | 5.4.1 | CSV ingestion (BOM, quoted commas, mixed types) |
| Alpine.js   | 3.14.1| reactive UI binding |

Everything else is custom JS in the single inline `<script>`: distribution functions (`NumLib`), regression math (`Stats`), diagnostics (`Diag`), Plotly factories (`Plots`), interpretation rules engine (`Interpret`), report exports (`Report`), tutorial overlay (`Tutorial`), self-test harness (`SelfTest`), and the Alpine `app()` factory.

---

## Architecture

The build went through 15 incremental, commit-sized steps tracked in `PLAN.md`. Each step left the app in a working state. The single file is organised top-to-bottom as:

```
<head>
  Pico CSS link
  ~400 lines custom CSS (theme tokens, grid layout, tabs, pills, plot containers, modal scaffolding, print CSS)
<body>
  <main x-data="app()">       Alpine root: header, sidebar, tabbed main, footer
  Modals + tutorial overlay
  CDN script tags
  <script>                    ~5 000 lines, organised as IIFE namespaces:
    SAMPLES                   8 inline reference datasets (453 rows total)
    NumLib                    erf, normCDF/Inv, gammaLn, betaInc, gammaInc, tCDF, fCDF, chi2CDF, helpers
    Stats                     buildDesign, fitOLS (QR), fitLogistic (IRLS), tInverse (bisection)
    Diag                      leverage, Cook's, VIF, DW, BP, JB, SW (Royston 1992), HC3
    Plots                     13 Plotly figure factories + theme + LOWESS smoother
    Interpret                 15 rules + composer + headline
    Report                    JSON / Markdown / window.print()
    Tutorial                  spotlight overlay + 11+4 step lists + popover positioning
    SelfTest                  NIST Norris / Longley / Wampler1 fixtures + R reference + dist fn tests
    app()                     Alpine factory — state + method dispatch
```

---

## Files

- `index.html` — the entire app (~5300 lines, ~270 KB; ~6 MB CDN libs cached on first load)
- `PLAN.md`   — original architecture spec
- `README.md` — this file

---

## License

MIT. Sample datasets are public-domain or used under their original licenses (Anscombe 1973, Henderson & Velleman 1981 mtcars, Longley 1967 NIST, R airquality 1973 EPA, R cars Ezekiel 1930, Fisher 1936 iris, UCI Pima Indians Diabetes).

The Boston housing dataset has been intentionally **not** included due to documented sourcing concerns; the King County housing subset is the modern replacement.
