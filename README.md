**By:** (your name here)

*A data science project for DSC 80 at UC San Diego.*

## Introduction

In professional *League of Legends*, minute 15 is roughly when the laning phase ends and the game opens up. This project asks: **how much is a gold lead at 15 minutes worth?** If the gold difference at 15:00 already tells us who will win, the second half of a pro game mostly plays out an advantage built early. That matters for how teams draft and plan their early game, and for how fans and analysts read a game while it's being played.

The data comes from [Oracle's Elixir](https://oracleselixir.com/), which publishes match data for professional games. The 2026 file covers January 1 to November 19, 2026: **119,316 rows** describing **9,943 games** across 44 leagues. Each game has 12 rows (five players and one team summary per side). This project uses the **19,886 team rows**.

| Column | Description |
|---|---|
| `gameid` | Unique ID of the game (two team rows share one `gameid`) |
| `league` | League the game was played in (e.g. LCK, LPL, LEC) |
| `side` | `Blue` or `Red` side of the map |
| `result` | `1` if this team won, `0` if it lost |
| `golddiffat15` | This team's total gold minus the opponent's total gold at 15:00 |
| `xpdiffat15`, `csdiffat15` | Experience and creep-score difference at 15:00 |
| `killsat15`, `deathsat15` | Kills and deaths at 15:00 |
| `gamelength` | Game length in seconds |
| `datacompleteness` | `complete` or `partial` (some leagues don't report minute-by-minute stats) |
| `void_grubs` | Number of void grubs (an early-game objective) the team took |
| `pick1` – `pick5` | The five champions the team drafted |

## Data Cleaning and Exploratory Data Analysis

**Cleaning steps**

1. **Kept only team rows.** The question is about a team's gold lead, and player rows would count each game six times per side.
2. **Kept only the relevant columns.** Many player-level columns (like `champion`) are empty for team rows.
3. **Converted 0/1 columns to booleans.** `result` and `playoffs` became `bool`. The `first*` objective columns have some missing values, so they use pandas' nullable `boolean` type, which keeps missing values missing instead of turning them into `True` or `False`.
4. **Parsed `date` as a timestamp** and converted `gamelength` from seconds to minutes.
5. **Added `gold_bucket`**, which splits `golddiffat15` into seven ranges from "down 3k+" to "up 3k+".
6. **Checked the data generating process:** the two team rows of a game mirror each other (one team's +1,000 is the other's −1,000), and `golddiffat15` sums to 0 within every game.

Rows with a missing `golddiffat15` (mostly "partial" games from a few leagues) were kept for the missingness analysis and left out only where an analysis needs the value.

| league   | side   | result   |   golddiffat15 | gold_bucket    |   gamelength_min |
|:---------|:-------|:---------|---------------:|:---------------|-----------------:|
| LIT      | Blue   | False    |           1107 | up 0.5k-1.5k   |             29.3 |
| LIT      | Red    | True     |          -1107 | down 0.5k-1.5k |             29.3 |
| LIT      | Blue   | True     |          -1260 | down 0.5k-1.5k |             37.4 |
| LIT      | Red    | False    |           1260 | up 0.5k-1.5k   |             37.4 |
| LIT      | Blue   | False    |           -223 | even (±500)    |             30.5 |

**Univariate analysis**

<iframe
  src="assets/gold-diff-hist.html"
  width="650"
  height="450"
  frameborder="0"
></iframe>

The gold difference at 15:00 is perfectly symmetric around 0, because every team's lead is its opponent's deficit. Most games are within about ±2,000 gold at 15 minutes, but the tails reach past ±10,000, which is a one-sided stomp.

**Bivariate analysis**

<iframe
  src="assets/winrate-by-gold-lead.html"
  width="650"
  height="450"
  frameborder="0"
></iframe>

Win rate climbs steadily with the size of the lead: about 50% when even at 15:00, 64% when up 0.5k–1.5k, and 92% when up 3k+.

**Interesting aggregates**

Win rate by gold lead at 15:00 in the six major leagues:

| gold_bucket    |   CBLOL |   LCK |   LCP |   LCS |   LEC |   LPL |
|:---------------|--------:|------:|------:|------:|------:|------:|
| down 3k+       |   0.077 | 0.099 | 0.085 | 0.1   | 0.106 | 0.108 |
| down 1.5k-3k   |   0.239 | 0.266 | 0.284 | 0.2   | 0.313 | 0.186 |
| down 0.5k-1.5k |   0.356 | 0.344 | 0.398 | 0.359 | 0.38  | 0.362 |
| even (±500)    |   0.5   | 0.5   | 0.5   | 0.5   | 0.5   | 0.5   |
| up 0.5k-1.5k   |   0.644 | 0.658 | 0.607 | 0.641 | 0.62  | 0.638 |
| up 1.5k-3k     |   0.761 | 0.728 | 0.712 | 0.8   | 0.687 | 0.814 |
| up 3k+         |   0.923 | 0.91  | 0.915 | 0.9   | 0.894 | 0.892 |

A 15-minute lead is worth about the same in every major league. Each one climbs from roughly an 8–11% win rate when down 3k+ to roughly 89–92% when up 3k+, so the value of an early lead looks like a property of the game itself rather than of any one region's playstyle.

## Assessment of Missingness

*Coming soon.*

## Hypothesis Testing

Teams that lead in gold at 15 minutes seem to convert that lead slightly better on blue side, so I tested whether the gap is bigger than chance would produce. Only teams that were ahead at 15:00 are included, which is one team per game.

- **Null hypothesis:** Among teams leading in gold at 15 minutes, blue-side and red-side teams win at the same rate. Any observed difference is due to random chance.
- **Alternative hypothesis:** Among teams leading in gold at 15 minutes, blue-side teams win at a higher rate than red-side teams.
- **Test statistic:** win rate of leading blue-side teams minus win rate of leading red-side teams. The alternative has a direction, so a signed difference is the right choice.
- **Significance level:** 0.05.

Leading blue-side teams won 76.7% of the time and leading red-side teams won 72.0%, an observed difference of 0.046. Across 5,000 permutations that shuffled the side labels, no simulated difference was that large, so **p < 0.0002** and I **reject the null hypothesis**. The data are consistent with blue-side teams converting early leads more often. This is an observational comparison, so it can't show that the side itself causes the difference.

<iframe
  src="assets/blue-red-permutation.html"
  width="650"
  height="450"
  frameborder="0"
></iframe>

## Framing a Prediction Problem

**Can we predict the winner of a game using only what we know at 15:00?**

- **Type:** binary classification. The response variable is `result` (win or loss).
- **Metric:** accuracy. Every game has exactly one winner and one loser, so the classes are perfectly balanced (50/50), and a false positive costs the same as a false negative here.
- **Information at the time of prediction:** only the state of the game at 15:00 (gold, experience, creep score, kills and deaths at 15) plus pre-game information (side, league, and the champions each team drafted). Full-game statistics such as total kills, towers, barons and game length are excluded because they're only known after minute 15.
- **Train/test split:** by game. The two team rows of a game mirror each other, so both are always on the same side of the split (80% of games for training, 20% for testing).

## Baseline Model

The baseline is a logistic regression in a single `sklearn` `Pipeline` with two features:

- `golddiffat15` (quantitative), used as-is.
- `side` (nominal), one-hot encoded into a single `side_Red` column.

It reaches **74.4% accuracy on the test set** (74.5% on the training set, so it isn't overfitting). That's far better than a 50% coin flip, but it's essentially the simple rule "the team ahead at 15 wins", which scores 75.0% on the same test set. The model only knows that more gold means more wins. The final model will add information about the kind of game each team is playing, starting with the champions each team drafted.

## Final Model

*Coming soon.*

## Fairness Analysis

*Coming soon.*
