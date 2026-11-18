# GTA VI Launch-Day Twitch CCU Forecaster

[![make.fun · gta6-twitch-ccu-forecast-01](https://make.fun/badge/gta6-twitch-ccu-forecast-01.svg)](https://make.fun/bounty/gta6-twitch-ccu-forecast-01?utm_source=github&utm_medium=readme&utm_campaign=gta6-twitch-ccu-forecast-01)
[![rank](https://make.fun/badge/gta6-twitch-ccu-forecast-01/jepebrutal030-web/gta6-twitch-ccu-forecast-01/rank.svg)](https://make.fun/bounty/gta6-twitch-ccu-forecast-01/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=gta6-twitch-ccu-forecast-01)
[![reward](https://make.fun/badge/gta6-twitch-ccu-forecast-01/jepebrutal030-web/gta6-twitch-ccu-forecast-01/reward.svg)](https://make.fun/bounty/gta6-twitch-ccu-forecast-01?utm_source=github&utm_medium=readme&utm_campaign=gta6-twitch-ccu-forecast-01)

Forecast how many people watch **Grand Theft Auto VI** live on Twitch in each hour
of its release day, 2026-11-19 (UTC), after the first hour's count is known.
Entry for the [make.fun competition](https://make.fun/bounty/gta6-twitch-ccu-forecast-01)
($250, 20 places, lowest mean pinball loss on the log scale wins).

## How it works

`main.py` (Python 3.13, standard library only, no network, deterministic) reads one
case file `/input/<id>.in` and writes `/output/<id>.out` with five quantiles
(5%, 25%, 50%, 75%, 95%) of Twitch concurrent viewers for each of hours 2..24.

The model is fit **at run time from the case's own `comparables`** (past big game
launches' hourly Twitch curves), so it adapts to whatever launches the case ships:

1. **Size-conditioned regression.** For each hour, regress
   `log(viewers_h / viewers_1)` on `log(viewers_1)` across comparables. Big hour-1
   launches (huge pre-launch hype, already near peak) grow little; small hour-1
   launches explode. The regression captures that slope.
2. **Console rolling-midnight shape.** Median log-ratio of the console-only
   rolling-midnight launches (GTA VI's own unlock pattern: local midnight, country
   by country). This carries the real unlock dynamics: overnight dip (hours 2-5),
   US East unlock at hour 6, US West at hour 9, US-evening peak at hours 19-22.
3. **Blend.** Median log-ratio = average of (1) and (2).
4. **Spread.** Half the regression's residual standard deviation at that hour
   (floored 0.20, capped 0.80 log units) — narrower than the raw residual, which is
   inflated by extreme small-launch outliers. Quantiles:
   `q_tau = p1 * exp(m_h + z_tau * s_h)`.

See [MODEL.md](MODEL.md) for the full derivation, assumptions, and validation
numbers.

## Validation (leave-one-out on the kit's 37 comparable launches)

Mean pinball loss, shipped code vs the house persistence baseline:

| Regime | This model | House baseline |
|---|---|---|
| Hour-1 > 50k (n=20) | **0.237** | 0.251 |
| Console rolling (n=6) | **0.292** | 0.484 |
| Big + console rolling (n=4, GTA VI's regime) | **0.202** | 0.289 |
| All comparables (n=37) | **0.581** | 0.930 |

(House baseline scores 0.387 on the practice cases; the practice answers are not
public, so the comparable leave-one-out is the honest local proxy.)

## Run it

```sh
mkdir -p out
MAKEFUN_INPUT_DIR=<kit>/input MAKEFUN_OUTPUT_DIR=out python3 main.py
# writes out/p00.out ... out/p03.out, byte-identical across runs
```

## Files

- `main.py` — the entrypoint (the only program file).
- `MODEL.md` — model derivation, assumptions, validation.
- `LICENSE` — MIT.

## License

MIT — see [LICENSE](LICENSE).
