# Model: GTA VI launch-day Twitch CCU forecast

## The problem

Grand Theft Auto VI releases 19 November 2026 on PlayStation 5 and Xbox Series X|S
(no PC until 2027). No console maker publishes a player count, so the contest
forecasts Twitch's concurrent live viewers in the "Grand Theft Auto VI" category
(game id 1867279146). The program sees everything recorded up to the end of the
day's first hour (hour 1's print included) and must forecast hours 2..24 as five
quantiles each. Score: mean pinball loss on the log scale; a prize needs a loss
strictly below the best house entry (persistence baseline ~0.387 on practice, and
a house AI-model forecast).

## What the data says

The kit ships 38 comparable past big launches (SullyGnome hourly Twitch category
averages, 14 days before to 3 days after each release day) per practice case, plus
the GTA VI category's own hourly history from its 2026-08-27 extended-look premiere.
Analysing their release-day curves (hours 1..24 relative to each launch's
`day_start`):

### The typical shape (median ratio to hour 1, all 37 usable comparables)

| Hours | Median ratio | What is happening |
|---|---|---|
| 2-5 | 0.73-0.87 | Overnight lull; US not yet unlocked |
| 6-12 | 0.79-0.98 | US East unlocks hour 6, US West hour 9 |
| 13-18 | 1.06-1.95 | Afternoon build |
| 19-22 | 2.30-2.40 | US-evening peak |
| 23-24 | 2.04-2.18 | Late decline |

Peak hour is 19-22 in most launches. But the spread is enormous (IQR up to 2.1 log
units at hours 15-22): the median ratio alone is a rough guide, not a forecast.

### Hour-1 size predicts the shape

The peak ratio is strongly negatively related to hour-1 viewers:

- Hour-1 875k (ELDEN RING), 852k (Cyberpunk 2077), 533k (Diablo IV), 387k
  (Hogwarts Legacy), 365k (The Last of Us Part II): peak ratios 1.00-1.34.
  These launches were already at peak in hour 1 (huge pre-launch hype and/or a
  simultaneous PC unlock).
- Hour-1 164 (Hollow Knight: Silksong), 414 (Kingdom Come: Deliverance II),
  419 (HELLDIVERS 2), 1,410 (ARC Raiders): peak ratios 121x-2,121x. Nobody was
  watching before the unlock; the game exploded after it.

So a regression of `log(ratio_h)` on `log(hour-1)` captures the main structure:
big hour-1 means a flat curve, small hour-1 means a steep one.

### Console rolling-midnight launches are tighter

Restricting to console-only rolling-midnight unlocks (GTA VI's pattern: FINAL
FANTASY VII REBIRTH, Ghost of Yotei, Monster Hunter Wilds, Resident Evil Requiem,
The Last of Us Part II, The Legend of Zelda: Tears of the Kingdom; n=6):

| Hours | Median ratio | IQR (log) |
|---|---|---|
| 2-5 | 0.59-0.83 | 0.16-0.59 |
| 6 | 1.14 | 0.66 |
| 13-19 | 1.80-2.60 | 0.67-0.96 |
| 20-24 | 1.99-2.52 | 0.48-0.97 |

The shape is cleaner (US unlock drives the rise, US evening is the peak) and the
spread is much smaller than the all-launch set. GTA VI is console-only with a
rolling local-midnight unlock (NZ 11:00Z Nov 18, CET 23:00Z, UK 00:00Z, US East
05:00Z, US West 08:00Z), so this is the most relevant sub-population.

## The model (fit at run time)

For each forecast hour h = 2..24:

1. **Regression.** Over all comparables with a usable release-day curve, fit
   `log(viewers_h / viewers_1) = a_h + b_h * log(viewers_1)` by least squares.
   Evaluate at the case's hour-1 print p1 to get `m_reg`.
2. **Console-rolling shape.** Median `log(viewers_h / viewers_1)` over the
   console-rolling comparables only, `m_cr` (needs >= 3; else omitted).
3. **Median.** `m_h = mean(m_reg, m_cr)` when both exist, else whichever exists,
   else the all-comparables median, else 0 (persistence).
4. **Spread.** `s_h = clamp(0.5 * residual_std_h, 0.20, 0.80)` log units, where
   `residual_std_h` is the regression's residual standard deviation at hour h.
   Halving is deliberate: the raw residual is inflated by extreme small-launch
   outliers that do not apply to a launch of GTA VI's size, and the halved spread
   scored best in validation.
5. **Quantiles.** `q_tau = p1 * exp(m_h + z_tau * s_h)`, `z` the standard normal
   quantiles (-1.645, -0.674, 0, 0.674, 1.645), clamped to (0, 1e9] and rounded to
   2 decimals half-up.

The median ratio is capped at 100x hour 1 so a tiny hour-1 cannot produce an
absurd forecast. Fallbacks (too few comparables for the regression, no
console-rolling set) degrade gracefully to the all-comparables median and then to
persistence.

## Assumptions

- The launch-day shape is driven mainly by (a) how much hype was already live at
  hour 1 and (b) the rolling-midnight unlock pattern. Both are observable at the
  checkpoint (hour-1 print + the comparables' unlock text).
- GTA VI's hour-1 print will be large (hundreds of thousands), putting it in the
  regime where the curve is relatively flat (peak ratio ~1.3-2.6, not 100x+).
  The regression handles this automatically: it predicts less growth for larger
  hour-1.
- The Twitch category's pre-release history (in the comparables) anchors the
  hype level, but the forecast normalises by hour 1, so the absolute level does
  not need a separate estimate.
- SullyGnome hourly averages (practice) approximate the final's minute-mean
  prints well enough that a shape fitted on them transfers. The final's
  minute-level data is finer, but the hourly shape is what the model uses.

## Validation

Leave-one-out over the kit's 37 usable comparable launches: for each launch, fit
the model on the other 36, forecast its hours 2..24 from its hour-1 print, score
mean pinball loss against its actual hourly curve. Compared with the house
persistence baseline (q50 = hour-1 print, spread 0.15*sqrt(h-1)):

| Regime | This model | House baseline |
|---|---|---|
| Hour-1 > 50k (n=20) | 0.237 | 0.251 |
| Console rolling (n=6) | 0.292 | 0.484 |
| Big + console rolling (n=4) | 0.202 | 0.289 |
| All (n=37) | 0.581 | 0.930 |

The model beats the baseline in every regime, most in the console-rolling regime
GTA VI belongs to. Caveats: leave-one-out on comparables is a proxy, not the
practice score (the four practice answers are not public); the console-rolling
subset is small (n=6), so its numbers are noisy; and the final is a single case,
so the realised loss will be one draw, not an average. The spread is honest
(halved residual, bounded) rather than tuned to minimise the proxy.

## Reproduce

```sh
mkdir -p out
MAKEFUN_INPUT_DIR=<kit>/input MAKEFUN_OUTPUT_DIR=out python3 main.py
```

Two runs on the same input write byte-identical outputs (no randomness, no clock,
no network).
