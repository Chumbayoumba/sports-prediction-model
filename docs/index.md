---
title: Sports prediction model — methodology and backtests
description: How Prognoz.win predicts football, cricket, basketball, hockey and esports — 10,000 simulations per match, tested on 90,000+ games.
---

# Sports prediction model: 10,000 simulations per match, tested on 90,000+ games

Open methodology and walk-forward backtests behind **[Prognoz.win](https://prognoz.win/en/)** — free daily predictions for football, cricket, basketball, ice hockey and esports in seven languages ([English](https://prognoz.win/en/) · [हिन्दी](https://prognoz.win/hi/) · [বাংলা](https://prognoz.win/bn/) · [Português](https://prognoz.win/pt/) · [Español](https://prognoz.win/es/) · [Français](https://prognoz.win/fr/) · [Русский](https://prognoz.win/)).

This repository documents **how the probabilities are produced and how well they hold up on past data**, including where the model loses to the betting market. Every number below comes from out-of-sample, walk-forward tests: the model only sees matches played before the one it predicts.

## Contents
1. [Why not just use bookmaker odds?](#why-not-just-use-bookmaker-odds)
2. [Football model](#football-model)
3. [Ice hockey, basketball and esports](#ice-hockey-basketball-and-esports)
4. [Cricket model](#cricket-model)
5. [From probabilities to a pick](#from-probabilities-to-a-pick)
6. [Accumulators](#accumulators)
7. [What did not work](#what-did-not-work)
8. [Reading the numbers honestly](#reading-the-numbers-honestly)
9. [Live predictions](#live-predictions)

## Why not just use bookmaker odds?
Bookmaker prices already contain a lot of information, so the market is the benchmark every model has to be compared with. After removing the bookmaker margin (Shin's method), the closing market is very hard to beat. Our goal is different: an **independent estimate** of each outcome that can be explained (ratings, form, rest, attack and defence strength) and then **checked against the market** — when the two disagree by more than 15 percentage points the pick is flagged and kept out of the "most likely" list.

## Football model
**Data:** 51,612 league matches from 18 European leagues, seasons 2012–2026 (results, shots, shots on target, corners, cards, pre-match prices).

**Features (55 in total), all computed sequentially without look-ahead:**
- margin-aware Elo rating with home advantage (K ≈ 18.9, home ≈ 92 Elo points) and a league-strength offset for cup ties;
- exponentially weighted attack and defence (goals, shots, shots on target, corners) with a half-life of ≈ 4.1 matches;
- form, rest days, congestion, table position and points gap;
- pre-match market probabilities (opening prices, never closing) as one of the inputs.

**Model:** two gradient-boosted Poisson regressors predict expected goals for each side; a classifier estimates the 1X2 probabilities directly; the two views are blended and temperature-calibrated on a hold-out season. Score probabilities come from a **Monte Carlo simulation of 10,000 matches** using a gamma-Poisson (over-dispersed) goal process, which gives every market at once: 1X2, double chance, totals, handicaps, both teams to score and exact score.

**Walk-forward results (51,612 out-of-sample matches):**

| Metric | Our model | Elo only | Market (de-vigged) |
|---|---|---|---|
| 1X2 accuracy | 50.2% | 49.9% | 51.3% |
| Log-loss (lower is better) | 1.0034 | 1.0092 | 0.9887 |
| Brier score | 0.5998 | 0.6037 | 0.5899 |
| Ranked probability score | 0.2045 | 0.2062 | 0.1999 |

With pre-match market prices added as features the model improves to **51.0% accuracy and 0.9945 log-loss**, still slightly behind the de-vigged market — which is exactly why the market is kept as a sanity check.

**How often the main pick wins, by confidence:**

| Our probability | Hit rate | Picks |
|---|---|---|
| ≥ 60% | 70.9% | 39,908 |
| ≥ 70% | 74.5% | 20,675 |
| ≥ 80% | 84.5% | 1,414 |

Over/under 2.5 goals: accuracy 56.4%, log-loss 0.6802.

## Ice hockey, basketball and esports
- **Ice hockey (NHL, KHL):** Poisson GLM on rating-adjusted goals plus a simulation of overtime and shoot-outs, so "winner including overtime" is priced correctly.
- **Basketball (NBA, EuroLeague):** ridge regression on rating, pace and rest; the margin follows a normal distribution, which prices spreads and totals.
- **Esports (CS2, Dota 2, League of Legends):** logistic model on Elo with tournament tier; series length (Bo1/Bo3/Bo5) is taken into account.

| Competition | Matches | Winner accuracy | Log-loss | Hit rate at ≥70% (n) | Hit rate at ≥80% (n) |
|---|---|---|---|---|---|
| NHL (ice hockey, winner incl. OT) | 4,811 | 57.6% | 0.6751 | 70.0% (377) | 81.6% (38) |
| NBA (basketball) | 4,595 | 65.7% | 0.6198 | 79.3% (1,388) | 83.5% (424) |
| EuroLeague (basketball) | 1,081 | 65.6% | 0.6158 | 77.7% (354) | 91.7% (108) |
| Counter-Strike 2 | 21,383 | 63.2% | 0.6405 | 76.9% (4,862) | 82.5% (1,224) |
| Dota 2 | 5,303 | 66.1% | 0.6229 | 75.5% (2,099) | 80.9% (879) |
| League of Legends | 6,534 | 65.4% | 0.6236 | 76.1% (2,429) | 81.8% (1,037) |

## Cricket model
Format-specific Elo ratings (Test/first-class, ODI/List A, T20) trained on **18,127 men's matches** from the open [Cricsheet](https://cricsheet.org) archive, with home advantage for internationals and a base draw rate for multi-day games. Fixtures come from public schedules; predictions are published with fair odds (1 / probability).

| Format | Matches (2019+) | Winner accuracy | Log-loss | Hit rate at ≥70% (n) |
|---|---|---|---|---|
| T20 / T20I | 6,260 | 60.0% | 0.6585 | 78.8% (641) |
| ODI / List A | 1,694 | 58.3% | 0.6666 | 74.1% (348) |
| Test / first-class (decisive results) | 1,050 | 62.3% | 0.6521 | 77.5% (138) |

## From probabilities to a pick
1. All candidate bets for a match are priced from the simulation.
2. The **main pick** is the outcome with the best balance of probability and price (favourite to win, or double chance when the favourite is not strong enough).
3. A **safer** alternative (higher chance) and a **bolder** one (higher odds) are shown next to it.
4. Picks where our probability differs from the de-vigged market by more than 15 points are excluded from "most likely" and accumulators.
5. Every pick is logged with a timestamp and a hash before kick-off and settled automatically after the match; [results are public](https://prognoz.win/statistika/), including losses and the reason a pick failed.

## Accumulators
The daily accumulator combines 3–5 legs with at least 70% probability each from different matches (no correlated legs) and targets total odds around 2.5; the "big-odds" version targets ~5.0 with legs of 55%+. The combined probability is the product of leg probabilities and is always shown next to the odds.

## What did not work
- **Geomagnetic activity, weather and "head-to-head" history** added nothing measurable to log-loss once ratings and form were in the model.
- **Head-to-head** is mostly noise beyond what ratings already capture.
- **Pure Dixon–Coles on goals alone** was worse than the de-vigged market.

## Reading the numbers honestly
- A 75% probability means the pick should lose about one time in four. Over a long run, calibration matters more than any single result.
- Accuracy on favourites is not the same as profit: short odds need a very high hit rate to break even.
- Betting involves risk. 18+ only, and only stake what you can afford to lose.

## Live predictions
- Today's predictions: [English](https://prognoz.win/en/) · [Hindi](https://prognoz.win/hi/) · [Bengali](https://prognoz.win/bn/) · [Portuguese](https://prognoz.win/pt/) · [Spanish](https://prognoz.win/es/) · [French](https://prognoz.win/fr/) · [Russian](https://prognoz.win/)
- Cricket: [prognoz.win/en/cricket/](https://prognoz.win/en/cricket/) · Football: [prognoz.win/en/football/](https://prognoz.win/en/football/) · Accumulator of the day: [prognoz.win/en/accumulator/](https://prognoz.win/en/accumulator/)

---
Text and tables: CC BY 4.0. Data sources: football-data.co.uk, Cricsheet (ODC-BY), ESPN public schedules, bo3.gg, EuroLeague.
