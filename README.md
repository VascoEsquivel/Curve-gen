# PeakForge — Curve Fit Studio

A single-file spectral peak/curve-fit plotting tool, modelled on the Renishaw WiRE
curve-fit panel. No install, no server, no dependencies — open `index.html` in any
browser.

## The peak model

Linear pseudo-Voigt, the same convention WiRE uses:

```
y(x) = H · [ m·G(x) + (1−m)·L(x) ],     m = %Gaussian / 100
G = exp(−4ln2·(x−x₀)²/w²)               L = 1 / (1 + 4(x−x₀)²/w²)
```

Both components have FWHM = `w` exactly, so the mixture does too. The area is
analytic, never numerically integrated:

```
A = H·w·[ m·√(π/4ln2) + (1−m)·π/2 ]
```

This was validated against real WiRE output (all 10 curves of a carbon Raman fit)
and reproduces every reported area to within 0.0006%. Numbers from this tool are
directly comparable to the instrument's.

## What it does

**Axes** — label, units, min, max, major increment, minor tick count, per axis.
Reversible X for XPS/IR. Auto-scaling Y.

**Baseline** — none / constant / linear / quadratic, over the normalised window
`u = (x−Xmin)/(Xmax−Xmin)`, so `c₁` and `c₂` read as "counts across the full plot".
Optionally refined during fitting.

**Peaks** — position, height, FWHM, and % Gaussian per component. The envelope
(sum) draws solid; components draw dashed underneath, optionally shaded. Negative
heights are supported (dip components).

**Labels** — a Label cell per row in the table; the text floats above that peak on
the plot with a leader line, and can be dragged anywhere.

**Table** — Curve Name, Centre, Width (FWHM), Height, % Gaussian, Type, Area,
Area %, Label. Every numeric cell is live-editable. Type auto-resolves to
Gaussian / Lorentzian / Mixed from the % Gaussian value.

## Beyond the reference tool

- **Synthetic data** — generate a realistic noisy spectrum from the current model
  (Poisson shot noise, or Gaussian absolute / % of max), seeded for reproducibility.
- **Import** — 2-column CSV / TSV / whitespace text; headers and `#`/`;` comments skipped.
- **Auto-find peaks** — robust MAD noise estimate plus a topographic prominence
  test, so it seeds real bands instead of harvesting noise ripples.
- **Auto-fit** — Levenberg–Marquardt with per-parameter trust bounds. Statistical
  (1/√y) or uniform weighting. Reports reduced χ², R², RMSE.
- **Uncertainties** — 1σ errors from the covariance matrix `χ²ᵣ·(JᵀJ)⁻¹`, plus a
  warning when two parameters are near-degenerate (|ρ| > 0.98).
- **Fix column** (C W H G) — freeze any individual parameter so it is excluded
  from the fit. This is the remedy when the report flags a degeneracy.
- **Residual panel** under the plot, sharing the X axis.
- **Analysis** — height ratio, area ratio, Δcentre and spectrum centroid for any
  two curves (I_D/I_G, I_2D/I_G, …).
- **Export** — 3× PNG, vector SVG, CSV of every component column, tab-separated
  table to clipboard, and JSON project save/load.
- Undo/redo, presets (Raman carbon, overlapping bands, XPS C1s, XRD Kα doublet),
  dark UI, collapsible panels.

## Reading the fit report

The report leads with **fit quality**, not χ². That is deliberate.

Reduced χ² is only ≈ 1 when your weighting model matches the true noise. In
randomised testing it reached **396 for a fit accurate to 0.8%**, purely because
unit weights were applied to data with a different noise scale. R² is no better —
it read **0.42 on a fit whose centres were correct to 0.3%**, because the data was
noise-dominated. Both numbers routinely say "bad" about good fits.

Fit quality compares the residual to the noise level measured from your own data,
window by window, so it is independent of the weighting choice:

| value | meaning |
|---|---|
| < 1.15 | at the noise floor — nothing left to fit |
| 1.15 – 1.6 | good, slight structure remaining |
| 1.6 – 3 | under-fitted, real structure remains |
| > 3 | the model does not describe this data |

When a region is much worse than the rest, the report names the x position —
that is usually where a component is missing. Tested by deleting a known peak
and checking the pointer: **8/8, located to within a third of a FWHM.**

The report also flags **undersampled** peaks (fewer than 4 points across the
FWHM) and components whose height is **within 2σ of zero** — the latter are
probably not real, and deleting them usually improves the fit.

## Validation

The peak model is checked against real WiRE output (above). The fitter is checked
by round-trip: generate a spectrum from known parameters, add noise, corrupt the
starting guess, fit, and compare against truth.

100 randomised trials per run, spanning 1–6 peaks, x-domains from `0.5–2.0` to
`10000–12000`, intensities over five orders of magnitude, all four baseline
models, three noise models, both weighting choices, and starting guesses
corrupted by up to 50%.

Current results:

- 0 exceptions, 0 non-finite parameters, 0 collapsed components
- median centre error 0.06 FWHM, median height error 14%
- 99/100 fits reach the noise floor; 0 false "poor" verdicts
- runtime: median 121 ms, worst 521 ms

Bugs this testing caught, none of which were visible by inspection:

1. **Sign error** in the Gauss–Newton update — every step went uphill and fits
   died at iteration 0.
2. **No step control** — components collapsed to zero width or fled the window
   from a poor starting guess.
3. **Colliding peak ids** — a counter read off a half-built state object, so
   selection silently targeted the wrong row.
4. **Derivative clamped at bounds** — a peak at exactly %Gaussian 0 or 100 got a
   zero derivative and could never leave that value. Pure Lorentzian and pure
   Gaussian starting points were permanently frozen.
5. **Noise estimated globally** — biased under shot noise, which mislabelled the
   four most accurate fits in 100 as "poor". Now estimated per window.

One thing testing ruled *out*: large x offsets. Fits at offset 0 and offset 10⁸
are bit-identical, so no special handling is needed.

## A warning worth reading

A low χ² does **not** mean the components are right. Heavily overlapped bands are
mathematically degenerate: many different combinations of position, width and
height produce the same envelope. In testing, an XPS 4-component fit reached
χ²ᵣ = 1.005 — a perfect fit to the noise floor — while individual component
heights were off by hundreds of percent.

This is why the uncertainties exist. If a component's height error is comparable
to the height itself, or the report flags a near-degenerate pair, the envelope is
not constraining that component. Fix a parameter from physical knowledge
(a known width, a known %Gaussian, a fixed peak separation) and refit.

## Controls

| | |
|---|---|
| double-click plot | add a peak there |
| drag ■ apex | centre + height |
| drag ▪ flank | FWHM |
| ← → / ↑ ↓ | nudge centre / height (Shift = 10×) |
| Ctrl+Z / Ctrl+Y | undo / redo |
| Del | delete selected curve |
| F | run auto-fit |
