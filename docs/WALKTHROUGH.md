# Arvyo: the full walkthrough

One real TESS sample, every stage of a detection, and the data contract that lets eleven repositories stay independent. The short version is in the [README](../README.md).

## How a detection actually happens

Every figure below is generated from one real sample — **TIC 138644215**, sector 40, 19,611 points at two-minute cadence over a 28.2-day baseline, with `crowdsap = 0.44`, meaning **more than half the light in the aperture is not from the target star**. That last number is why this is a hard case rather than a demo.

Nothing is cherry-picked or hand-edited. `scripts/smoke_run.py --make-docs-figures --seed 42` regenerates all of it from a fresh clone.

### 1 · The input

A full TESS sector at two-minute cadence — dense enough that individual transits are invisible by eye at this scale.

![A dense raw TESS light curve spanning 28 days, appearing as a thick noisy band with no obvious dips](images/pipeline-01_raw_lightcurve.jpg)

### 2 · The representation

201-bin global and 81-bin local phase-folded views — the model's actual input, not the raw series.

![Global and local phase-folded views of the light curve, the transit dip now clearly visible in both](images/pipeline-02_views.jpg)

### 3 · The period search

One dominant, clean peak at **1.545 days, SDE 16.8** — more than double the 7.0 detection gate. It recovers the sample's own recorded period of 1.54493 d to within **0.013%**.

![A TLS periodogram with a single sharp dominant peak and no aliasing harmonics](images/pipeline-03_tls_periodogram.jpg)

### 4 · The fold

![The light curve phase-folded at 1.545 days, with the binned global view overlaid, showing a clean symmetric transit](images/pipeline-04_phase_fold.jpg)

### 5 · The four hypotheses — and the part that matters

All four forward models are fitted to the same data. `best_explanation` picks **planet**, matching the true label.

**But it very nearly didn't, and the reason is the most interesting thing in this repository.** At `crowdsap = 0.44`, the blend model's free `dilution` parameter can track the planet model almost exactly. On raw chi² the two were a near-tie — 19726.59 against 19726.71, a difference of 0.12 out of ~19,700. A different noise draw or random seed would flip that winner.

So ranking is by **BIC**, not chi²: chi² penalised by `n_params × log(n_points)`. Blend pays for its extra free parameter, and the comparison becomes robust instead of a coin flip. The same planet/blend degeneracy is pinned down in `tests/test_end_to_end.py` so it cannot silently regress.

### 6 · Posteriors

Period, duration and depth marginals from `fit_emcee`, seeded at the period search's answer. Tight marginals reflect both a clean fit and an accurately recovered period.

![Corner-style posterior marginal distributions for period, duration and depth, each tightly peaked](images/pipeline-06_posteriors.jpg)

### 7 · Vetting — including the number that doesn't flatter the result

Odd/even and secondary-eclipse checks at the recovered ephemeris.

Odd-transit depth (0.0060) comes out **visibly deeper** than even-transit depth (0.0043) — a difference of 1.70e-3. That is real per-transit scatter on a single, moderately diluted sector. It is also exactly the signature a genuine eclipsing-binary false positive would show, **so this number alone does not clear the target.** It is the secondary-eclipse check — depth ratio ~0.003, no secondary detected — together with the transit shape that argues for a planet here.

![Odd versus even transit depth comparison and a secondary-eclipse search window showing no detection](images/pipeline-07_vetting.jpg)

### 8 · The classes, side by side

All five committed samples, including a `blend` drawn from **real Kepler DR25 centroid-offset data** (KIC 6974867) rather than a simulation.

![Phase-folded curves for planet, eclipsing binary, blend, starspot and null classes arranged in a gallery](images/pipeline-08_class_gallery.jpg)

---

## The data contract

Two halves of the project, built to be worked on independently, meet at exactly one interface — a frozen `.npz` schema in `arvyo/contract.py`:

```python
SCHEMA_VERSION = "1.0"

REQUIRED_ARRAYS = ["time", "flux", "flux_err"]
OPTIONAL_ARRAYS = ["flux_raw"]          # present for starspot/null classes
REQUIRED_META   = ["tic_id", "label", "sector"]
OPTIONAL_META   = ["period_days", "epoch_btjd", "crowdsap", "mission",
                   "augmented", "injection_params"]
LABELS          = ["planet", "eb", "blend", "starspot", "null", "unknown"]
```

Any change to it requires bumping `SCHEMA_VERSION` and updating **both** repos' READMEs. `arvyo-pipeline` never writes into `arvyo-data`; it only reads.

The same discipline governs the glue layer. `worker.py` and `run.py` never import `foldr`, `fitr` or `localizr` internals — they invoke the published CLIs and parse stdout JSON and exit codes. `fitr`'s output is embedded verbatim under `model_fit`, never reshaped. Exit codes carry meaning: `0 → clear`, `3 → ambiguous`, `4 → no_significant_signal`.

And there is a short circuit that saves most of the compute: if the period search's SDE and SNR both fall under the gate, the target returns `no_period` immediately and **`fitr` is never run**.
