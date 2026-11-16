**By Wei Yen**

---

## Introduction

Top lane is often called "the island." It is the longest lane on the map, it is far from the dragon pit, and junglers spend the least time there. Many top laners know the feeling: you win your lane, and the game is still lost somewhere else. This project uses professional match data to ask:

**How much does winning top lane matter, compared to the other roles?**

The data comes from [Oracle's Elixir](https://oracleselixir.com/) and covers every tracked professional League of Legends game from 2022: **150,228 rows** from **12,519 games** across 55 leagues. Each game has 12 rows: one for each of the 10 players, plus one summary row for each team.

The columns most relevant to this question:

| Column | Description |
|---|---|
| `gameid`, `league`, `side` | Which game, which league, and which side (Blue/Red) the row belongs to |
| `position` | Role: `top`, `jng`, `mid`, `bot`, `sup`, or `team` |
| `champion` | Champion played |
| `result` | 1 if the player's team won, 0 otherwise |
| `golddiffat15`, `xpdiffat15`, `csdiffat15` | Gold / experience / creep-score lead over the opposing player in the same role at 15 minutes |
| `killsat15`, `deathsat15`, `assistsat15` | Kills, deaths and assists at 15 minutes |
| `kills`, `assists`, `teamkills` | End-of-game kills and assists for the player, and total kills for their team |
| `damageshare` | Share of the team's damage to champions dealt by this player |
| `gamelength` | Game length (seconds) |

---

## Data Cleaning and Exploratory Data Analysis

**Cleaning steps:**
- **Kept only player rows.** Team rows summarize the same game, so mixing them with player rows would count each game twice and mix two different kinds of rows.
- **Converted `result` to a boolean**, since it only records win/loss.
- **Converted `gamelength` from seconds to minutes.**
- **Added `kill_participation`** = (kills + assists) / team kills, which measures how involved a player was in their team's fights. Games with zero team kills get `NaN` instead of dividing by zero.
- **Added `lane_state`**, which groups `golddiffat15` into *Behind* (≤ −1,000), *Even*, and *Ahead* (≥ +1,000). Because `golddiffat15` compares two players in the same role, it directly measures who won the lane. 1,000 gold (about two kills) is a common rule of thumb for a clear lane win.
- **Left missing 15-minute values as missing.** They are missing for entire leagues (see the missingness section), so filling them in would invent lane results for games where we have none.

The first few rows of the cleaned data (selected columns):

| gameid                | league   | side   | role      | champion   | result   |   golddiffat15 |   kill_participation |
|:----------------------|:---------|:-------|:----------|:-----------|:---------|---------------:|---------------------:|
| ESPORTSTMNT01_2690210 | LCKC     | Blue   | Top       | Renekton   | False    |        391.000 |                0.444 |
| ESPORTSTMNT01_2690210 | LCKC     | Blue   | Jungle    | Xin Zhao   | False    |        541.000 |                0.889 |
| ESPORTSTMNT01_2690210 | LCKC     | Blue   | Mid       | LeBlanc    | False    |       -475.000 |                0.556 |
| ESPORTSTMNT01_2690210 | LCKC     | Blue   | Bot (ADC) | Samira     | False    |       -793.000 |                0.444 |
| ESPORTSTMNT01_2690210 | LCKC     | Blue   | Support   | Leona      | False    |        443.000 |                0.778 |

### Univariate Analysis

<iframe src="assets/top_golddiff_hist.html" width="800" height="500" frameborder="0"></iframe>

Top laners' gold leads at 15 minutes are centered at 0 (every top laner's lead is the other top laner's deficit) and are roughly symmetric. Most lanes are within ±2,000 gold, but many games still have a clear lane winner, which gives us plenty of won lanes to study.

<iframe src="assets/top_champions.html" width="800" height="500" frameborder="0"></iframe>

Gnar, Gwen and Ornn were the most-played top laners of 2022. The list mixes carries (Gwen, Camille, Aatrox, Jayce) with tanks (Ornn, Sejuani, Gragas). This matters for our question, because tank top laners are often picked to *absorb* pressure rather than to win the game alone.

### Bivariate Analysis

<iframe src="assets/winrate_by_golddiff.html" width="800" height="500" frameborder="0"></iframe>

For every role, a bigger gold lead at 15 minutes means a higher win rate, but **the top lane curve is the flattest**. A top laner who is 2,000 gold ahead wins noticeably less often than a mid laner or jungler with the same lead, and a top laner who is far behind loses less often. The result of top lane swings the game less than the result of any other lane.

<iframe src="assets/kp_by_role.html" width="800" height="500" frameborder="0"></iframe>

Top laners have the lowest kill participation of any role: a median of about 54%, compared with 67–70% for every other role. They are present for fewer of their team's fights, which is one way an early lead could fail to spread to the rest of the map.

### Interesting Aggregates

Team win rate (%) by role and lane state at 15 minutes:

| role      |   Behind (-1k) |   Even |   Ahead (+1k) |   Ahead minus Behind |
|:----------|---------------:|-------:|--------------:|---------------------:|
| Top       |           28.6 |   50.0 |          71.5 |                 42.9 |
| Jungle    |           22.1 |   50.0 |          77.9 |                 55.9 |
| Mid       |           18.2 |   50.0 |          81.8 |                 63.6 |
| Bot (ADC) |           23.9 |   50.0 |          76.1 |                 52.2 |
| Support   |           23.9 |   50.0 |          76.1 |                 52.2 |

Each lane has one player on each team, so the *Behind* and *Ahead* columns mirror each other, and *Even* is exactly 50%. The last column shows how much winning versus losing a lane changes the team's chance of winning. It is smallest for top lane, at **42.9 percentage points**, compared with 52–56 for bot, support and jungle and **63.6** for mid.

---

## Assessment of Missingness

### NMAR Analysis

The 15-minute columns (`golddiffat15`, `xpdiffat15`, `killsat15`, …) are missing for about 15% of rows. We **do not believe these columns are NMAR**. Whether a value is missing has nothing to do with the value itself: a big gold lead is no more likely to go unrecorded than a small one. Instead, it depends on **where the game was played**. The 15-minute stats come from each game's minute-by-minute timeline. For the Chinese leagues (LPL and LDL) and a few smaller events, Oracle's Elixir only receives end-of-game statistics from mid-2021 onward, so the timeline-based columns are blank. That makes the missingness **MAR**, because it depends on another observed column, `league`.

A column that *could* plausibly be NMAR is `split`, which is missing for leagues that don't divide their season into splits. To tell whether that missingness depends on the value itself, we would want more information, such as each league's official season format.

### Missingness Dependency

We tested whether the missingness of `golddiffat15` depends on other columns, using one row per team per game (the top laner's row).

<iframe src="assets/missing_by_league.html" width="800" height="500" frameborder="0"></iframe>

**`league`:** The null hypothesis is that the distribution of `league` is the same whether or not `golddiffat15` is missing. We used the total variation distance (TVD) as the test statistic and a significance level of 0.05. The observed TVD was **0.99**, and none of the shuffled TVDs came close, giving a **p-value of 0.0**. We reject the null: **missingness depends on `league`.**

<iframe src="assets/missing_league_perm.html" width="800" height="500" frameborder="0"></iframe>

**`kills` (the top laner's end-of-game kills):** The null hypothesis is that the mean of `kills` is the same whether or not `golddiffat15` is missing. We used the absolute difference in means as the test statistic and a significance level of 0.05. The observed difference was 0.015 kills, with a **p-value of about 0.71**. We fail to reject the null: there is no evidence that **missingness depends on `kills`**, which fits the explanation that the data is missing because of its source, not because of anything that happened in the game.

<iframe src="assets/missing_kills_perm.html" width="800" height="500" frameborder="0"></iframe>

Because missingness depends on `league`, dropping these rows removes the LPL and LDL. Our 15-minute conclusions apply to leagues with timeline data, not to Chinese pro play.

---

## Hypothesis Testing

We looked at every player who **won their lane by 1,000+ gold at 15 minutes** and compared how often their team went on to win.

- **Null hypothesis:** Top laners who are 1k+ gold ahead at 15 minutes win at the same rate as players in other roles who are 1k+ gold ahead.
- **Alternative hypothesis:** Top laners who are 1k+ gold ahead at 15 minutes win at a *lower* rate than players in other roles who are 1k+ gold ahead.
- **Test statistic:** (win rate of players in other roles who are ahead) − (win rate of top laners who are ahead). The alternative is one-sided, so we used a signed difference.
- **Significance level:** 0.05.

Under the null, a player's role has nothing to do with whether their lead turns into a win, so we ran a permutation test that shuffles the role labels.

<iframe src="assets/hypothesis_test.html" width="800" height="500" frameborder="0"></iframe>

Top laners who were 1k+ gold ahead won **71.5%** of their games, compared with **77.9%** for players in other roles who were 1k+ ahead, a gap of 6.4 percentage points. None of the 2,000 shuffled gaps was that large, so the **p-value is below 0.0005** and we **reject the null hypothesis**. The data is consistent with the idea that an early top lane lead turns into a win less often than an early lead in other roles.

This doesn't *prove* that top lane matters less. Players in the same game aren't independent (when the top laner is ahead, the jungler often is too), and this is observational data, so other differences between roles could explain the gap.

---

## Framing a Prediction Problem

**Using only the top laner's information at 15 minutes, can we predict whether their team wins?**

- This is **binary classification**: the response variable is `result` (win/loss).
- If top lane strongly decided games, the top laner's 15-minute state should predict the result well. How accurately it does measures how much top lane alone tells us about the outcome.
- **Information at time of prediction:** the prediction is made at the 15-minute mark, so the model only uses what is known by then: the top laner's gold/XP/CS leads, kills, deaths and assists at 15, their champion, their side, and the league. End-of-game stats are not used.
- **Metric: accuracy.** Every game has one winning and one losing top laner, so the classes are exactly 50/50, and a wrong prediction is equally bad in either direction. Random guessing would score 50%.
- Rows without 15-minute data (mostly LPL/LDL) are dropped, since their features don't exist. That leaves 21,262 top laners, split 75% / 25% into training and test sets.

---

## Baseline Model

The baseline is a logistic regression in a single sklearn `Pipeline`, with three features:
- `golddiffat15` (quantitative): the top laner's gold lead at 15 minutes, used as is.
- `xpdiffat15` (quantitative): the top laner's experience lead at 15 minutes, used as is.
- `side` (nominal: Blue/Red): one-hot encoded, dropping one column.

There are no ordinal features.

The baseline reaches **62.2% test accuracy** (61.7% on the training set, so it isn't overfitting). That's clearly better than guessing, but it isn't a *good* model: the top laner's lane result alone leaves most of the outcome unexplained. Given our question, that is itself an interesting finding.

---

## Final Model

**Features added:**
- `csdiffat15`, `killsat15`, `deathsat15`, `assistsat15` (standardized): gold lead alone doesn't show *how* the lane was won. A lead from farming differs from a lead from kills, and deaths show whether the top laner has been feeding the enemy team.
- **Engineered `early_kp`**: the share of the team's kills in the first 15 minutes that the top laner took part in. Top laners have the lowest kill participation of any role, and a top laner who joins early fights is less "on an island," so their lead is more likely to spread to the rest of the map.
- **Engineered `kda15`**: (kills + assists) / (deaths + 1) at 15 minutes, summarizing how the top laner's fights have gone.
- `champion` (one-hot): the same lead means different things on different champions. Tanks like Ornn are picked to support the team, while champions like Renekton or Jayce are picked to snowball a lead.
- `league` (one-hot): leagues differ in play style and in how often leads turn into wins.

**Model and tuning:** We kept logistic regression, because win probability rises smoothly with lead size (see the win rate plot above). We tuned it with 5-fold `GridSearchCV` on the training set over:
- `C`, the inverse regularization strength. With hundreds of one-hot champion columns, some regularization is needed to avoid overfitting to rarely played champions.
- `min_frequency` of the one-hot encoder. Champions and leagues with fewer games than this are grouped into a single "infrequent" category, another guard against overfitting.

The best hyperparameters were **C = 0.01** and **min_frequency = 100**. A random forest (tuning `max_depth` and `min_samples_leaf`) did slightly worse in cross-validation.

<iframe src="assets/gridsearch.html" width="800" height="500" frameborder="0"></iframe>

**Result:** Using the same train/test split, test accuracy rose from **62.2% (baseline) to 64.8% (final)**. The gain comes from using more of what the top laner did in the first 15 minutes. Even so, the top laner's whole early game correctly predicts only about 65% of games, which again suggests that top lane alone rarely decides a professional game.

---

## Fairness Analysis

**Does the final model work equally well for Blue-side and Red-side top laners?**

- **Group X:** Blue side. **Group Y:** Red side.
- **Evaluation metric:** precision, i.e. when the model predicts a win, how often it is right.
- **Null hypothesis:** The model is fair. Its precision for Blue-side and Red-side top laners is roughly the same, and any difference is due to random chance.
- **Alternative hypothesis:** The model is unfair. Its precision differs between Blue side and Red side.
- **Test statistic:** absolute difference in precision. **Significance level:** 0.05.

<iframe src="assets/fairness.html" width="800" height="500" frameborder="0"></iframe>

Precision was **65.4%** for Blue side and **65.2%** for Red side, with a **p-value of about 0.93**. We fail to reject the null hypothesis: there is no evidence that the model's precision differs between sides, although this doesn't prove the model is perfectly fair.

---

**Conclusion:** Winning top lane matters, since a top laner who is 1k+ gold ahead wins about 72% of the time. It matters *less* than winning any other lane, though, and the top laner's early game alone predicts only about 65% of outcomes. For top laners in solo queue, the takeaway is to take the lead to the rest of the map rather than staying on the island.
