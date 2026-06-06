# Predicting Player Roles in Professional League of Legends

**Author:** Flandre Yu

This project was completed for DSC 80 at UC San Diego.

## Introduction

This project uses the **2025 League of Legends esports match dataset from Oracle's Elixir**. The original dataset contains **120,456 rows and 165 columns**. Each professional match contributes both player-level and team-level rows. I focus on player-level rows and investigate the following question:

> **How do post-game statistics differ across in-game positions, and how accurately can a player's position be predicted from those statistics?**

This question matters because each League of Legends position has a distinct strategic responsibility. Top laners, junglers, mid laners, bot laners, and supports differ in farming, vision control, damage contribution, resource use, and participation in fights. Understanding these statistical patterns helps explain how team roles appear in real match data.

The most relevant columns are:

- `position`: the player's in-game role; the prediction target.
- `cspm`: creep score per minute, measuring farming rate.
- `vspm`: vision score per minute, measuring vision contribution.
- `damageshare`: the player's share of the team's champion damage.
- `dtaken share`: the player's share of the team's damage taken.
- `KDA`: `(kills + assists) / deaths`, with zero deaths replaced by one.
- `KP`: kill participation, `(kills + assists) / team kills`.
- `monster pm`: monster kills per minute.
- `controlwards per min`: control wards purchased per minute.
- `result`: whether the player's team lost (`0`) or won (`1`).

## Data Cleaning and Exploratory Data Analysis

### Data Cleaning

I first removed team-level rows because the analysis and prediction task concern individual player positions. I also restricted the data to rows labeled `complete`, since partial match records are missing several important early-game and player-level statistics.

I then engineered several role-related features:

- `dtaken share`: player damage taken per minute divided by the team's total damage taken per minute.
- `KP`: `(kills + assists) / team kills`.
- `KDA`: `(kills + assists) / deaths`.
- `controlwards per min`: control wards bought divided by game length in minutes.
- `monster pm`: monster kills divided by game length in minutes.

After selecting the relevant columns, the cleaned player-level dataset contains **92,210 rows** and has no missing values in the selected columns.

Here is the head of the cleaned DataFrame:

| gameid           | league   | position   |   result |   KDA |    KP |   damageshare |   dtaken share |   vspm |   cspm |   monster pm |
|:-----------------|:---------|:-----------|---------:|------:|------:|--------------:|---------------:|-------:|-------:|-------------:|
| LOLTMNT03_179647 | LFL2     | top        |        0 | 1     | 0.667 |         0.402 |          0.234 |  0.641 |  8.819 |        0     |
| LOLTMNT03_179647 | LFL2     | jng        |        0 | 0.333 | 0.333 |         0.099 |          0.312 |  1.093 |  5.389 |        4.975 |
| LOLTMNT03_179647 | LFL2     | mid        |        0 | 0.5   | 0.333 |         0.278 |          0.126 |  0.754 |  7.877 |        0     |
| LOLTMNT03_179647 | LFL2     | bot        |        0 | 0.667 | 0.667 |         0.138 |          0.165 |  0.377 |  9.46  |        0.452 |
| LOLTMNT03_179647 | LFL2     | sup        |        0 | 0.667 | 0.667 |         0.083 |          0.163 |  3.09  |  1.432 |        0     |

### Univariate Analysis

The CSPM distribution is bimodal. The large group near one CSPM is mostly composed of support players, while the broader group around seven to nine CSPM represents farming roles. This indicates that farming rate is likely to be useful for distinguishing player positions.

<iframe
  src="./graph/cspm_distribution.html"
  width="100%"
  height="550"
  frameborder="0"
></iframe>

### Bivariate Analysis

CSPM differs substantially by position. Bot, mid, and top players generally have high CSPM, jungle players have a lower farming rate because they obtain resources from jungle monsters, and support players have by far the lowest CSPM. The overlap among top, mid, and bot also suggests that CSPM alone will not perfectly distinguish all five positions.

<iframe
  src="./graph/cspm_by_position.html"
  width="100%"
  height="550"
  frameborder="0"
></iframe>

### Interesting Aggregates

The following pivot table compares average post-game statistics by position and game result:

| position   |   KDA (loss) |   KDA (win) |   cspm (loss) |   cspm (win) |   damageshare (loss) |   damageshare (win) |   dtaken share (loss) |   dtaken share (win) |
|:-----------|-------------:|------------:|--------------:|-------------:|---------------------:|--------------------:|----------------------:|---------------------:|
| bot        |        2.23  |      10.244 |         8.801 |        9.247 |                0.265 |               0.273 |                 0.14  |                0.132 |
| jng        |        1.847 |       9.694 |         5.969 |        6.691 |                0.163 |               0.17  |                 0.3   |                0.327 |
| mid        |        2.032 |       9.36  |         8.193 |        8.578 |                0.261 |               0.255 |                 0.169 |                0.162 |
| sup        |        1.735 |       8.483 |         1.024 |        1.027 |                0.083 |               0.078 |                 0.152 |                0.143 |
| top        |        1.456 |       7.253 |         7.513 |        7.91  |                0.228 |               0.224 |                 0.24  |                0.236 |

Winning players have much higher KDA values than losing players across every position, showing that KDA is strongly affected by game outcome. In contrast, role-specific statistics such as CSPM and damage shares retain meaningful differences across positions, which makes them useful for role prediction.

## Assessment of Missingness

### NMAR Analysis

A column that may be **NMAR** is `xpdiffat25`. This statistic can only be recorded if a game reaches 25 minutes. Games that end before 25 minutes are often highly one-sided, so the unobserved 25-minute XP difference might have been unusually large had the game continued. In that sense, the probability that `xpdiffat25` is missing may be related to the unobserved value itself.

However, observed variables such as `gamelength`, `xpdiffat15`, and `xpdiffat20` may partly explain this missingness. Additional detailed game-state information shortly before the game ended could potentially make this missingness explainable by observed data and therefore MAR rather than NMAR.

### Missingness Dependency

I analyzed the missingness of `xpat15`, an early-game XP statistic. I used total variation distance (TVD) to compare categorical distributions between rows where `xpat15` is missing and rows where it is observed.

For the first permutation test:

- **Null hypothesis:** The missingness of `xpat15` does not depend on `league`.
- **Alternative hypothesis:** The missingness of `xpat15` depends on `league`.
- **Test statistic:** TVD between the league distributions of missing and non-missing rows.
- **Result:** The observed TVD was approximately **0.991**, and none of 300 permutations produced a statistic at least this large, giving a simulated p-value of approximately **0**.

I therefore reject the null hypothesis. The missingness of `xpat15` appears to depend strongly on league, likely because different leagues have different data-collection coverage.

For the second test:

- **Null hypothesis:** The missingness of `xpat15` does not depend on `position`.
- **Alternative hypothesis:** The missingness of `xpat15` depends on `position`.
- **Result:** The simulated p-value was **1.0**.

I fail to reject the null hypothesis, so there is not enough evidence that `xpat15` missingness depends on player position. Overall, the missingness of `xpat15` is more consistent with **MAR** than NMAR because it is associated with an observed variable, league.

<iframe
  src="./graph/xpat15_missingness_by_league.html"
  width="100%"
  height="600"
  frameborder="0"
></iframe>

## Hypothesis Testing

I tested whether mid laners have a higher average vision score per minute than top laners.

- **Null hypothesis:** Mid and top players have the same average VSPM.
- **Alternative hypothesis:** Mid players have a higher average VSPM than top players.
- **Test statistic:** Mean VSPM for mid players minus mean VSPM for top players.
- **Significance level:** 0.05.
- **Observed statistic:** Approximately **0.080**.
- **Simulated p-value:** Approximately **0**, because none of 1,000 shuffled samples produced a statistic at least as large as the observed statistic.

I reject the null hypothesis. The data provide strong evidence that professional mid players have a higher average VSPM than professional top players. A one-sided difference in means is appropriate because the alternative hypothesis specifically predicts a higher mean for mid players.

<iframe
  src="./graph/vspm_hypothesis_test.html"
  width="100%"
  height="550"
  frameborder="0"
></iframe>

## Framing a Prediction Problem

The prediction task is to predict a professional player's in-game position from post-game statistics. This is a **multiclass classification** problem because the response variable, `position`, has five classes: top, jungle, mid, bot, and support.

The time of prediction is **after the game has ended**, so all selected post-game statistics are available. I use **accuracy** as the evaluation metric because the five position classes are approximately balanced, making accuracy a clear measure of the proportion of correctly classified players.

## Baseline Model

The baseline model is a `DecisionTreeClassifier` with a maximum depth of four. It uses four quantitative features:

- `cspm`
- `vspm`
- `dtaken share`
- `monster pm`

There are **four quantitative features, zero ordinal features, and zero nominal features**, so no categorical encoding is needed. The model and all processing steps are contained in a single scikit-learn `Pipeline`.

The data are split into a 75% training set and a 25% held-out test set. The model achieves:

- **Training accuracy:** 0.786
- **Test accuracy:** 0.779

The test accuracy is much higher than the 0.20 accuracy expected from uniform random guessing across five classes. This is a strong baseline, although it likely confuses roles such as mid and bot that share similar farming and damage patterns.

## Final Model

The final model is a `RandomForestClassifier`. It improves on the baseline by adding features that capture more role-specific behavior:

- `KP` captures involvement in team kills.
- `damageshare` measures offensive contribution relative to teammates.
- `controlwards per min` captures investment in vision.
- `log_KDA` reduces the influence of unusually large KDA values.
- `damage_dtaken_ratio` compares offensive contribution with damage absorbed.

The transformations are implemented inside a `ColumnTransformer`, which is included in the same scikit-learn `Pipeline` as the random forest.

Before tuning, I selected three hyperparameters because they control the complexity and stability of the forest:

- `n_estimators`: the number of trees.
- `max_depth`: the maximum depth of each tree.
- `min_samples_split`: the minimum number of samples needed to split a node.

I used `GridSearchCV` with four-fold cross-validation. The best-performing values were:

- `n_estimators = 150`
- `max_depth = 15`
- `min_samples_split = 5`

The final model achieves:

- **Training accuracy:** 0.918
- **Test accuracy:** 0.833

The test accuracy improves from 0.779 for the baseline model to 0.833 for the final model on the same held-out test set. The improvement suggests that combining multiple trees with additional role-specific features improves generalization to unseen player records.

## Fairness Analysis

I tested whether the final model has similar accuracy for players on winning and losing teams.

- **Group X:** Players on winning teams (`result = 1`).
- **Group Y:** Players on losing teams (`result = 0`).
- **Evaluation metric:** Accuracy.
- **Null hypothesis:** The model is equally accurate for winning-team and losing-team players, and the observed difference is due to random chance.
- **Alternative hypothesis:** The model's accuracy differs between winning-team and losing-team players.
- **Test statistic:** Absolute difference in group accuracy.
- **Significance level:** 0.05.

The observed results were:

- **Winning-team accuracy:** 0.837
- **Losing-team accuracy:** 0.830
- **Absolute difference:** 0.007
- **Permutation-test p-value:** 0.171

Since the p-value is greater than 0.05, I fail to reject the null hypothesis. The test does not provide evidence that the model is less accurate for one group than the other, so the model appears to have similar accuracy for players on winning and losing teams.
