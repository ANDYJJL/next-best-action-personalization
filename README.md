# Next-Best-Action Personalization — Pentathlon

> By Andy Liu
> All figures below are test-set estimates from the analysis notebook, not
> live A/B results. The full notebook is in `notebooks/pentathlon_nptb.ipynb`;
> the dataset itself is not included in this repo (see `data/README.md`).

## The problem: too many emails, and nobody agrees on who gets what

Pentathlon, a sporting-goods retailer, had a coordination problem. Its departments
each ran their own promotional emails, and customers were getting too many of
them. An earlier six-month randomized test on email frequency had settled one
argument: sending two emails per week beat one, three, or four on six-month
customer value, because heavier frequency drove unsubscribes that wiped out the
short-term revenue gain.

But the frequency cap created a second argument. With only two emails per
customer per week, departments now had to share a scarce slot — and they could
not agree on how to allocate it. The endurance director argued his emails should
get priority for women; the racquet director argued her category sold better to
women online than in stores. Everyone had segment reports about *who buys what*.
Nobody had evidence about *which message works on which customer*.

Then came the lucky accident. While departments argued, marketing had been
allocating the weekly email slot **at random** as an interim compromise. Random
allocation meant the company already sat on something close to experimental
data: customers assigned to different department messages — or to no message at
all — for no reason other than chance. No new six-month test was needed. The
question became: for each individual customer, which message (if any) makes
them most likely to buy, and which one makes the most money?

## The data

One row per customer–email pair, built from each customer's most recent email:

- **600,000 customers**, split 70/30 into train (420,000) and test (180,000)
- **7 arms**: six department messages (endurance, strength, water, team,
  backcountry, racquet) plus a no-message control group
- **Features**: age bucket, gender, income, neighborhood education level and
  average children, plus each department's purchase frequency over the past year
- **Outcomes**: `buyer` — clicked the email and purchased within two days
  (yes/no); `total_os` — total order size in euros, conditional on buying,
  across all departments, not just the one that sent the message

Because assignment was random, a model trained on this data can answer
counterfactual questions: *what would this customer have done under each
message?* That is the whole trick. No uplift model was used; the analysis stays
inside the next-product-to-buy (NPTB) framework.

## The solution: score every customer under every message, then follow the money

Two stages, repeated for each of four models:

**Stage 1 — Will they buy?** Fit a purchase-probability model
P(buyer = yes | customer, message), with message-by-purchase-history
interactions so the effect of a message can differ by customer. Four models
were tuned with cross-validation on at least two hyperparameters each:

- Logistic regression (pyrsm) with message × frequency interactions
- Neural network / MLP (hidden layers, alpha)
- Random forest (n_estimators, max depth, max features; CV AUC 0.877)
- XGBoost (max depth 3, 600 trees, learning rate 0.05; CV AUC 0.886)

**Stage 2 — If they buy, how much?** Fit an OLS regression on log order size
using buyers only, with message indicators and message-by-frequency
interactions, and back-transform to euros.

**Expected profit per customer-message pair:**

```
Expected profit = P(buy) × (1 − COGS) × E(order size | buy)
                = P(buy) × 0.40 × E(order size | buy)      # COGS = 60%
```

For every customer in the test set, all seven actions (six messages plus
no-message) were scored, and the action with the highest expected profit was
chosen. That is the personalized policy. It was then compared against three
baselines: the best single message sent to everyone, random assignment (the
status quo), and sending nothing.

## The results: personalization wins under every model

Expected profit per customer on the test set (€), and the total for a typical
5,000,000-customer blast:

| Model | Personalized | Best single message | Random (status quo) | No message |
|---|---:|---:|---:|---:|
| Logistic regression | 0.488 | 0.460 (endurance) | 0.409 | 0.312 |
| Neural network | **0.517** | 0.483 (endurance) | 0.427 | 0.316 |
| Random forest | 0.497 | 0.444 (water) | 0.404 | 0.329 |
| XGBoost | 0.483 | 0.455 (endurance) | 0.404 | 0.309 |

| Model | Personalized total | vs best single | vs random | vs no message |
|---|---:|---:|---:|---:|
| Logistic regression | €2,440,515 | +€140,645 (+6.1%) | +€396,190 (+19.4%) | +€879,510 (+56.3%) |
| Neural network | **€2,584,695** | +€171,640 (+7.1%) | +€451,795 (+21.2%) | +€1,003,620 (+63.5%) |
| Random forest | €2,483,080 | +€262,455 (+11.8%) | +€465,450 (+23.1%) | +€839,450 (+51.1%) |
| XGBoost | €2,415,150 | +€140,950 (+6.2%) | +€393,235 (+19.5%) | +€870,620 (+56.4%) |

Three things stand out:

1. **The ranking never changes.** For all four models: personalized > best
   single message > random > no message. The win is not a modeling artifact.
2. **The best-fitting model is not the most profitable one.** XGBoost had the
   highest cross-validated AUC, but the neural network's personalized policy
   earned the most money — a reminder to select on the business metric, not
   just fit.
3. **Personalization is genuinely heterogeneous, not a disguised
   one-size-fits-all.** Under the neural network's profit policy, endurance is
   optimal for 61% of customers, but team (14%), water (8%), strength (7%),
   backcountry (6%) and racquet (4%) each win real segments. Under the random
   forest the split is even wider: endurance 34%, water 20%, backcountry 19%,
   racquet 9%, strength 8%, team 7%, and no-message for 2% of customers — the
   model sometimes concludes the most profitable action is to send nothing at
   all. The simple logistic model, by contrast, collapses to endurance for
   ~88% of customers, which is a sign it is missing heterogeneity, not that
   heterogeneity is absent.

## The policy recommendation (and its critique)

The case also asked for a verdict on a draft policy: each month, give each
customer's top two messages a 50/50 split of that customer's email slots,
re-estimated from the first three weeks of data in the last week of the month.

It is a step in the right direction — it uses expected profit — but it has
four weaknesses:

- A forced 50/50 split is not profit-optimal. When the top message is far
  ahead of the second, the split leaves money on the table.
- It never considers the no-message option, even for customers whose predicted
  profit is low or negative — inviting fatigue and unsubscribes, the exact
  problem the frequency test had already exposed.
- Re-estimating on only the first three weeks of the month risks selection
  bias: customers contacted early in the month may behave differently.
- It has no monitoring: no guardrails for model drift or fairness across
  departments.

The improved version: allocate shares in proportion to predicted expected
profit (allowing 100% to the top message when the gap is large), add a
no-message threshold (send nothing below a small positive profit cutoff), and
track performance weekly against a holdout/control group so drift is caught
before the policy is updated.

## What I would do differently now

- Validate the profit ranking with a live holdout experiment before full
  rollout: test-set estimates assume the world stays still, and email fatigue
  is exactly the kind of effect that moves.
- Add calibration checks on the purchase probabilities: the profit ranking
  inherits every error in P(buy).
- Report uncertainty, not just point estimates — the gap between the top two
  policies is real but not enormous, and a decision-maker should know how
  much of it is noise.

## Stack

Python · Polars · pyrsm · scikit-learn (MLP, random forest) · XGBoost ·
Jupyter. Results reproduced from the notebook in this repo.

## Code

Core pipeline, cleaned up from the submitted notebook. The full notebook
additionally contains the per-model tuning grids, gains curves, and the
question-by-question write-up; the logic below is the part that produces every
number in the tables above.

### 1. Load and split

```python
import polars as pl

df = pl.read_parquet("data/pentathlon_nptb.parquet")  # 600,000 rows, 16 cols
train = df.filter(pl.col("training") == 1)            # 420,000
test  = df.filter(pl.col("training") == 0)            # 180,000

freq_cols = [c for c in df.columns if c.startswith("freq_")]
# freq_endurance, freq_strength, freq_water, freq_team, freq_backcountry, freq_racquet
ACTIONS = ["endurance", "strength", "water", "team",
           "backcountry", "racquet", "control"]       # 6 messages + no-message
```

### 2. Stage 1 — P(buy): logistic regression with message × history interactions

The interaction terms are the whole point: a message's effect is allowed to
differ by the customer's purchase history, which is what makes the
recommendation personal instead of one-size-fits-all.

```python
import pyrsm as rsm

def add_int_cols(data: pl.DataFrame) -> pl.DataFrame:
    exprs = []
    for m in ACTIONS:
        if m == "control":
            continue
        msg_ind = (pl.col("message") == m).cast(pl.Float64)
        for f in freq_cols:
            exprs.append((msg_ind * pl.col(f)).alias(f"int_{m}_{f}"))
    return data.with_columns(exprs)

train_int = add_int_cols(train)
evar = (["message", "age", "female", "income", "education", "children"]
        + freq_cols
        + [c for c in train_int.columns if c.startswith("int_")])

clf_logit = rsm.model.logistic(
    data={"train": train_int}, rvar="buyer", lev="yes", evar=evar)
```

The other three models use the same features (one-hot message/age/female,
standardized numerics) and are tuned by 3-fold cross-validation on AUC:

```python
# Neural network (sklearn MLP): hidden_layer_sizes x alpha
#   grid: (32,16), (32,32), (64,32) x 1e-5, 1e-4, 1e-3
# Random forest: n_estimators x max_depth x max_features
#   best: 400 trees, depth 15, max_features="sqrt" (CV AUC 0.877)
# XGBoost (pyrsm): max_depth x n_estimators x learning_rate
#   best: depth 3, 600 trees, lr 0.05 (CV AUC 0.886)
```

### 3. Counterfactual scoring: every customer under every action

```python
base = test.select(["custid", "age", "female", "income",
                    "education", "children"] + freq_cols)
cf = base.join(pl.DataFrame({"message": ACTIONS}), how="cross")
# cf has 180,000 x 7 rows: each customer scored under each message.
# Predict P(buy) for every row with the fitted model, then keep the
# best action per customer for the probability ranking (Q1/Q2).
```

### 4. Stage 2 — order size, then expected profit

Order size is modeled only on buyers (buyer == "yes"), on log1p scale, and
back-transformed to euros. Expected profit uses a 60% COGS, i.e. a 40% margin.

```python
# OLS on buyers only: log1p(total_os) ~ features + message + message x freq
# E_order_size = expm1(predicted log1p(total_os))
expected_profit = p_buy * 0.40 * expected_order_size
```

### 5. Policies compared on the test set

```python
# Personalized:  for each customer, the action with max expected profit
# Best single:   the one message with the highest mean expected profit,
#                sent to everyone (endurance for LR/NN/XGB, water for RF)
# Random:       mean expected profit across all 7 actions (the status quo)
# Control:      expected profit of the no-message action for everyone
# Totals:       per-customer profit x 5,000,000 for a typical blast
```
