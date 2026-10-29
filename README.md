# No Traffic Alert: redesigning an alert nobody read

This is one of the weekly alert went out to every account manager on the team. With it being live for more than 2 years, this one has less than 200 views and almost nobody acted on it.
This is the story of finding out why, building a replacement, and optimizing it into something people can actually act on.

> **Note:** This is a recreation of internal work using synthetically generated data. No real companies, people, accounts or figures are included. The dataset is produced by `data/generate_synthetic_data.ipynb`, which reproduces the statistical *shape* of the original problem (heavy-tailed revenue, batch-enabled dead integrations, intermittent outages).
> 

Three tabs: `1. Final`, `2. Rejected`, `3. Original`.

---

## The problem

Each client casino runs several game providers, and every provider on every casino should be producing traffic every day. When one stops, it could be a broken integration, a client-side change, or a line the client quietly turned off. Either way, somebody should find out why.

An alert existed to catch exactly this. It listed every casino × provider with zero traffic in three overlapping blocks (`3. Original`). It had been running for a long time, and the account managers (KAMs) receiving it had mostly stopped reading it. I set out to find out whether it still fits how they actually work.

This is the original dashboard:

<p align="center">
  <img src="https://github.com/chia-chang/no-traffic-alert/blob/main/images/3.%20Original/ZERO%20Traffic%20Alert%20(original).png" width="780"/>
</p>


## What the users said

I interviewed KAMs separately, and surprisingly none of them used it as an alert. Some of them used it as a clean up list, for the 

One described his real workflow:

> *"Every few months I go through all my lines and see which ones have no data and should be closed."*
> 

Another explained what would make him call the clients:

> *"If an important provider suddenly has no traffic, that's when we'd go and maybe adjust the lobby position or perhaps run promotion on it. If it is a small casino or a smaller provider, then we usually don’t act on it"*
> 

So the dashboard has two jobs: 

1. **close what is dead** (quarterly, problems that have been sitting for a while)
2. **call about what just went silent** (weekly, spot new issue timely before any potential loss continues). 

The original dashboard mixed them altogether and flooding them with clients the KAMs would not actually work on, so readers could not tell which rows asked for which action, and did neither.

---

## The final dashboard (`1. Final`)

<p align="center">
  <img src="https://github.com/chia-chang/no-traffic-alert/blob/main/images/1.%20Final/1.%20Final%20No%20Traffic%20Alert.png" width="930"/>
</p>

### **No traffic 50 days.**

The cleanup list: casinos silent for the whole window, with their go-live date, so they could actually identify if the casino has not traffic just because they were newly live or has actually been dead and haven't been contributing wager for a long time.

### **No traffic this week.**

One row per casino, only casinos where at least one provider had a full no traffic day this week *and the no traffic cost at least €1,000*. 

Cell color = how many days that provider was silent this week (1–7, one hue, darker means more days). 

Cell number = what those silent days cost at the provider's own normal daily rate. 

Row total ("Potential Wager") = the sum of the colored cells, and the table is sorted by it. 

"No traffic entire casino" = how many days the *whole* casino was dark. Blank means nothing is worth attention as of now.

<p align="center">
  <img src="https://github.com/chia-chang/no-traffic-alert/blob/main/images/how-to-read.png" width="930"/>
</p>

Note that the 1000 potential wager is decided based on statistics.

---

## My approach

### First instinct: Filter out some data → Refused.

My first move was subtraction. The page was numbing because it showed everything, so I filtered out the rows I thought nobody acted on. 

Feedback: every row on that page was there because someone had once needed it, and I was not allowed to take any of it away.

If nothing can be removed, the only way to make a page readable is to **rank** it, so I decided to put a number on each row that says how much it matters, and let the reader stop wherever their time runs out. That number became Potential Wager, and everything else on the final page follows from it: the sort order, the threshold, the row total that equals the sum of its cells.

### Second Attempt:  Categorized casino issues(`2. Rejected`) → Refused

I then split the page into four mutually exclusive tables: never live, under-performing, whole casino no traffic, at least one provider stopped. Each with its own action, plus a header stating
every window and threshold. Structurally it was right, no casino appeared twice, and the cleanup
population was separated from the incident population.

Feedback: *"too much stuff"*, *"a KAM should get it without reading the description"*.

These worksheets were helpful in a more analytical way instead of helping a KAM instantly know what to do next.

I kept all these in the workbook as rejection is always part of the work too.

---

## Decisions that mattered

- **Color means one thing.**
    
    The original colored cells nobody could act on. The final version uses one hue for days without traffic and leaves everything that needs no attention white.
    
- **Potential = daily wager rate × silent days, with a baseline that ignores silence.**
    
    Each provider's *average daily wager over the days it had traffic in the 43 days before this week*, times its zero-traffic days this week. 
    
    I had first replaced this with an "actual gap" (normal week − this week) because it is verifiable against the client's own numbers, and on production data the two agreed on
    median (ratio 1.00). The business sent it back for a good reason: a fixed four-week average shrinks week by week for any casino that has *stopped*, so the number fell exactly when it should have stayed loud. Averaging only over days with traffic fixes that, and lets a casino live for three days be measured on those three days instead of a 43 day average that is mostly zeros. 
    
- **Potential wager below €1,000 is filtered out.** 
For the smallest providers, data shows that the normal week to week swing is about €900, so anything smaller cannot be told apart from an ordinary week. Larger providers lose far more than €1,000 from a single no traffic day, so the same number is strict for small providers and sensitive for large ones.
    
    This number is set from 4 weeks of data and will be safer to re-check quarterly.
    `docs/design-decisions.md`.
    

---

## What changed

|  | `3. Original` | `1. Final` |
| --- | --- | --- |
| Structure | 3 blocks | 2 blocks |
| Ordering | completely alphabetical by casinos | € potential this week |
| Color | each a different concept | one hue = days silent; blank = no attention needed |
| Urgency Flag | none | Potential Wager per casino, sorted; row total = sum of its cells  |

---

## Repository

| Path | What it is |
| --- | --- |
| `No_Traffic_Alert_portfolio_final.twbx` | Packaged workbook with the synthetic data embedded. Opens on `1. Final`; `2. Rejected` and `3. Original` are the other tabs. |
| [`data/generate_synthetic_data.ipynb`](https://github.com/chia-chang/no-traffic-alert/blob/main/data/generate_synthetic_data.ipynb) | Generates the dataset (scenario-driven, seeded) and validates it in pandas. |
| [`data/traffic_monitor_synthetic.csv`](https://github.com/chia-chang/no-traffic-alert/blob/main/data/traffic_monitor_synthetic.csv) | 39,600 rows · 130 casinos · 8 providers · 50 days |
| [`docs/user-guide.md`](https://github.com/chia-chang/no-traffic-alert/blob/main/docs/user_guide.md) | A thorough guide for the account managers |
| [`docs/design-decisions.md`](https://github.com/chia-chang/no-traffic-alert/blob/main/docs/design_decisions.md) | Why each choice was made, what was rejected, the threshold analysis |
| [`docs/calculated-fields.md`](https://github.com/chia-chang/no-traffic-alert/blob/main/docs/calculated_fields.md) | Every calculated field on the shipped page, with Tableau order-of-operations notes |

## Technologies

- **Tableau**: nested FIXED level-of-detail expressions, parameter thresholds, context-filter order of operations, row-level security, email subscriptions
- **Python** (pandas, NumPy): synthetic data generation with a validation harness; threshold analysis on production data; workbook XML transformation with lxml
- **SQL / Athena:** grouping sets and provisioning tables during the investigation

---

made with ✨ + 🌿 + 💛 by Chia Chang
