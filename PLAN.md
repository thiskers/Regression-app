# Regression-app — Implementation Plan

## Context

Build a single-file standalone HTML app for **real statistical regression analysis** — not a teaching demo. The user can open `index.html` from disk, load a sample or upload their own CSV, fit a regression, see a full diagnostic suite (R-style), and read a plain-English auto-interpretation of what the model means. Bundled samples and an in-app guided tutorial cover first-time users; numerical accuracy validated against NIST StRD reference datasets covers serious analysts.

The tool must be defensible as "professional grade": coefficient inference (SE, p, CI), full assumption checks, logistic regression, and an interpretation engine that flags problems and suggests next steps.

## Stack (locked)

| Concern | Choice | Why |
|---|---|---|
| Math core | `ml-regression-multivariate-linear` + `ml-matrix` (CDN) | OLS via QR; matrix ops we build SE/diagnostics on |
| Plotting | Plotly.js (CDN) | Polished interactivity, native PNG export, all diagnostic plots |
| CSV | Papa Parse (CDN) | Robust to BOM, quoted commas, mixed types |
| UI shell | Alpine.js + Pico.css (CDN) | No build step; ~25 KB total; professional defaults |
| Build | None | One file, double-click to run |

**Custom code we own** (~1500 lines): all inferential statistics (none of the libs ship them) + interpretation rules + tutorial overlay + NIST self-test harness.

## Single-file structure (`index.html`)

```
<head>      Pico CSS (CDN) + ~200 lines custom CSS (overrides, print CSS)
<body>
  <main x-data="app()">         Alpine root: header, sidebar, tabbed main, footer
  <template>                    tutorial step, modals
  <script src="cdn/...">        ml-matrix, ml-regression, papaparse, plotly, alpine (deferred)
  <script>                      our JS (~1500 lines), organized as IIFE namespaces:
    SAMPLES        inline datasets
    NumLib         t/F/chi2/normal CDFs, inverse normal (Acklam)
    Stats          fitOLS (QR), fitLogistic (IRLS), buildDesign
    Diag           leverage, Cook's, VIF, DW, BP, JB, SW
    Plots          13 Plotly figure factories
    Interpret      RULES list + run() composer
    Report         JSON / Markdown / print-CSS
    Tutorial       spotlight overlay + 10 steps
    SelfTest       NIST validation harness
    app()          Alpine factory (state + methods)
```

## Models (v1)

- **Simple linear** (p=1): adds scatter+fit+CI+PI plot
- **Multiple linear** (OLS via QR)
- **Polynomial** (degree 2–6, via feature expansion)
- **Logistic** (binary, IRLS, with separation detection)

**Out of v1**: ridge/lasso, robust regression, Poisson, mixed effects, weighted LS, interactions UI (user can pre-compute), CV/train-test split.

## Custom statistics we implement

On top of `ml-matrix`:

- **OLS**: solve via QR (NOT normal equations — required to pass NIST Filip/Longley)
- **Coefficient inference**: SE from `(X'X)^-1 σ²`, t-stats, two-sided p via t-CDF, 95% CI, standardized β
- **Goodness of fit**: R², adj R², AIC, BIC, log-likelihood, RMSE
- **Overall test**: F-stat with p-value
- **Multicollinearity**: VIF per predictor
- **Residual diagnostics**: hat matrix diagonal (leverage), studentized residuals, Cook's distance
- **Assumption tests**: Durbin-Watson (autocorr), Breusch-Pagan (heteroscedasticity), Jarque-Bera + Shapiro-Wilk (normality)
- **HC3 robust SEs**: optional toggle for when BP fires
- **Distributions**: normal CDF (Cody erfc), inverse normal (Acklam), t-CDF / F-CDF / chi2-CDF via incomplete beta / lower incomplete gamma
- **Logistic**: IRLS loop (maxIter=50, tol=1e-8), null deviance, McFadden pseudo-R², ROC + AUC, calibration deciles, separation detection

## Sample datasets (bundled inline)

| Key | n × cols | Teaches |
|---|---|---|
| `anscombe` | 11 × 8 | Identical stats, different plots — why diagnostics matter |
| `mtcars` | 32 × 11 | Classic multiple regression with mild collinearity |
| `longley` | 16 × 7 | NIST numerical-accuracy stress test (cond ≈ 5×10⁹) |
| `kinghousing` | 50 × 6 | Real-world multiple regression w/ categorical + heteroscedasticity |
| `airquality` | 111 × 4 | Time-ordered (Durbin-Watson demo) + missing-data handling |
| `cars_speed_dist` | 50 × 2 | Heteroscedastic + nonlinear (polynomial demo) |
| `iris_binary` | 100 × 5 | Logistic with near-separation |
| `pima` | 200 × 8 | Logistic with realistic AUC ~0.83 |

(Boston housing dropped on sourcing/ethics grounds; King County subset is the modern replacement.)

## Diagnostic plot suite (Plotly)

1. Scatter + fit + 95% CI band + 95% PI band (simple linear only)
2. Residuals vs Fitted (with LOWESS overlay)
3. Normal Q-Q
4. Scale-Location
5. Residuals vs Leverage with Cook's contours
6. Histogram of residuals + normal overlay
7. Predictor correlation heatmap (p ≥ 2)
8. Partial-regression ("added variable") small-multiples (p ≥ 2)
9. Predicted vs Actual
10. ROC + AUC (logistic)
11. Calibration (logistic)
12. Coefficient forest plot (p ≥ 3)
13. Influence index plot (when influential points exist)

## Interpretation engine

Flat list of rules; each `{id, section, applies(model), severity, render(model)}`. Sections: **Overall fit / Predictor effects / Assumption issues / Recommendation**. A composer rule produces a one-line headline from the assembled output.

Rules cover: high R², low R², F-test failure, significant predictor (with effect size + CI), non-significant predictor, high VIF, BP-flagged heteroscedasticity, non-normal residuals, influential points, logistic separation, suspiciously high R² (overfit flag), Anscombe-style hidden nonlinearity.

Style: backticks for variable names; always quote effect + uncertainty; "no evidence" not "no effect"; warnings include actionable next step.

## Tutorial

Custom spotlight overlay (~150 lines, no third-party). Full-screen `<div>` with cutout via `box-shadow: 0 0 0 9999px rgba(0,0,0,0.5)` on the highlighted element. Step list:

1. Welcome → 2. Load mtcars sample → 3. Pick variables → 4. Click Fit → 5. Read Summary → 6. Read Coefficients → 7. Diagnostic tests → 8. Residuals vs Fitted plot → 9. Q-Q + Leverage → 10. Interpretation tab + "try Anscombe" CTA.

Branch tutorial for logistic introduces ROC + calibration + separation when family flips to `logistic`.

## Numerical accuracy (the credibility layer)

Hidden self-test page at `index.html#selftest`. Validates against NIST StRD certified values:

- **Norris, NoInt1, NoInt2** (well-conditioned): β + SE agree to ≥ 12 digits
- **Longley** (cond ≈ 5×10⁹): β agree to ≥ 6 digits
- **Filip** (degree-10 poly, cond ≈ 1×10¹⁰): β agree to ≥ 4 digits via QR
- **Wampler1** (perfect-fit): RSS rounds to 0
- Distribution functions vs R `pt`/`pf`/`pchisq`/`pnorm` at 20 points each

Renders pass/fail table; one-line summary at top ("ALL 47 TESTS PASS").

## Export

1. **JSON state dump** (full model payload, matrices serialized as `{__matrix__: rows}`)
2. **PNG figure export** (Plotly built-in modebar, 2× scale)
3. **Markdown report** (auto-interpretation + coef table as MD + base64 PNG figures)
4. **PDF** via `window.print()` + print CSS that hides chrome and inlines static SVG copies of figures

(No jsPDF/pdfmake — bundle bloat, worse output than OS print dialog.)

## Must-have features beyond core math

These separate "toy" from "professional"; all in v1:

- Missing-data handling: listwise (default) + mean/median impute, with dropped-rows notice
- Categorical encoding: auto-detect, dummy by default, reference-level chooser
- Variable transforms: log / sqrt / square / standardize per column
- Robust SE (HC3) toggle
- Predictions tab: single-row form + "predict on new CSV"
- Influence diagnostics with click-to-exclude refit
- Reproducibility metadata in JSON export (file SHA256, lib versions, timestamp)
- CSV parse-failure surfacing ("X column has 12% non-numeric values — treat as currency / percent / European decimal?")

## Out of scope (v1.1 candidates)

Interaction-terms UI, weighted LS, bootstrap CIs, k-fold CV, train/test split, Box-Cox finder, localStorage state save, stepwise selection (with warnings if added), ridge/lasso.

## Build order

1. Skeleton + Pico shell + sample loader + data preview + column-type inference
2. NumLib + Stats.fitOLS (QR) + Coefficients table + Summary tab
3. Diag (leverage/Cook/VIF/DW/BP/JB/SW) + all 9 OLS plots
4. Interpret rules engine + Tutorial overlay
5. Logistic (IRLS) + ROC/calibration + 2 logistic samples + polynomial expansion + Predictions tab
6. Export (print CSS + Markdown + JSON) + SelfTest + NIST validation
7. Polish: must-have gaps, gotchas, cross-browser pass

## Critical files

- `/home/user/Regression-app/index.html` — the entire app
- `/home/user/Regression-app/README.md` — run instructions, library versions, last NIST validation result

## Verification

**Numerical**: open `#selftest`, confirm all NIST + distribution tests pass. Spot-check mtcars `mpg ~ wt + hp + cyl` against R `summary(lm())` — β to 10+ digits, SE to 8+, R²/F to 8+.

**UX**: load each of 8 samples → fit produces interpretation + plots, no console errors. Anscombe set 4 must fire `influential_present` or `anscombe_pattern`. Iris-binary on Petal.Length alone must fire `logistic_separation`. Tutorial walkable end-to-end. CSV upload of a file with NAs, a string column, and a comma-decimal column produces clear feedback.

**Export**: print → PDF renders figures as static SVG, no chrome. JSON → reload-able. Markdown → renders in GitHub preview.

**Performance**: synthetic 10k × 20 fits in < 1.5 s; 100k × 5 produces a graceful "consider sampling" warning.

**Cross-browser**: Chrome, Firefox, Safari (incl. mobile Safari for the file picker and tutorial overlay).
