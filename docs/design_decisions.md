# Design decisions

The dashboard is a short list of judgement calls. This file records each one, the alternative that was considered, and the evidence. Figures come from four weeks of production data (anonymized here) and
from the synthetic dataset in this repository where noted.

## 1. One question per page

*Which casinos had providers with no traffic this week, what did it cost, who should call.*
Everything on the page serves that sentence. Two other questions were explicitly pushed out:

- **Declining but still running** (a big client 30% down, no zero days). Built, then removed: it doubled the color legend and made the dominant provider light up on nearly every row. Different question,
  different page.
- **Enabled but barely used** (a provider live at a casino but doing a few euros a week). That is a sales conversation, not an alert. The cleanup list covers it at 50 days.

## 1b. Nothing may be removed, so everything must be ranked

The first attempt at making the page readable was to filter rows out. The team refused: each row had
a reason to exist for someone. The constraint forced the design that shipped — if the population
cannot shrink, the page needs a single number per row that orders it (Potential Wager), a threshold
that is stated rather than hidden, and a layout that reads top-down so a reader can stop at any point
and have seen the most important part.

## 2. Row = casino, cell = provider

KAMs act per casino — one phone call — so a row is a casino. The provider grid inside the row tells them
what to say on that call. The first attempt (four routed tables, one per situation; `2 · Rejected` in the
workbook) was turned down by the business as "too much"; the second collapsed it to this single grid.

## 3. Color encodes one variable

| Version | Colors | Reviewer reaction |
| --- | --- | --- |
| 1 | red stopped · yellow down · green normal · gray never used | "I can't tell what each means" |
| 2 | 3-step ramp for days silent + a pale tint for "down but running" | "What are the two extra colors?" |
| shipped | 7-step single hue = days silent; blank = nothing to do | — |

Green and gray were removed on principle: every color in an alert should mean *do something*. "Nothing
to do" is white, whatever the reason — ran every day, not set up here, under the threshold, no baseline.
The guide spells those out once so nobody asks what the blank means.

## 4. Whole-casino silence gets its own column

"No traffic entire casino = N days" is computed at casino level (a day when all providers together had
zero wager). It does not depend on how many providers the casino has, and it is not inferred from the
row's colors — a row can be fully colored because each provider had its own silent days without the
casino ever being fully dark. The column removes that ambiguity instead of asking the reader to judge it.

## 5. Potential Wager = rate × silent days, baseline over days with traffic

Shipped: for each provider with ≥ 1 zero-traffic day this week,
*average daily wager over the days it had traffic in the 43 days before this week × zero-traffic days
this week*, summed per casino.

Three definitions were tried.

| | Baseline | Potential | Problem |
| --- | --- | --- | --- |
| Original | 50-day average, silent days included | avg × zero days | a casino quiet for weeks has a baseline diluted by its own silence |
| Iteration | average of the 4 full weeks before this week | normal week − this week (actual gap) | same dilution, one week slower; and a 2-week-dead casino's number shrinks every week until it vanishes — exactly when it should stay loud |
| **Shipped** | **43 days, days with traffic only** | **avg daily × zero days** | a line that ran on 2 days out of 43 is measured on those 2 |

The actual-gap version had real merits — the number is what the client would recognize, and on
production data its median ratio to the rate × days estimate was 1.00 — but the business asked for the
rate × days form specifically because of stopped casinos, and because a casino live for three days
should be measured on those three days, not on a 43-day average that is mostly zeros.

**Guard considered and rejected.** The known weakness (two traffic days × seven silent days = a number
the line never made) could be closed with "at least 14 baseline days with traffic". It was rejected on
purpose: it would also hide newly live casinos, which are the ones most likely to have an integration
problem in their first weeks. Instead the tooltip shows how many days the average rests on, and the
guide says so.

Both the rate × days potential and the "row total = sum of cells" property come for free from using
the same condition in all four fields; see `calculated-fields.md` §6.

## 6. Window = 43 days before this week

The source holds 50 days. This week takes 7; the rest is baseline. Not 4 fixed weeks (see §5), not
30 days (weekday mix drifts and the last 13 days are wasted). Because only days with traffic are
averaged, weekday alignment matters less than it did for the fixed-week version.

Known limit: a casino live for under a week has a baseline of a few days. Disclosed, not filtered.

## 7. The threshold — one parameter, €1,000

### 7.1 Why one and not two

An earlier version had two: *provider usually ≥ €1,000/week* (size) and *missing ≥ €1,000* (amount).
With potential = avg daily × zero days, and zero days ≤ 7, potential ≥ €1,000 implies
avg daily × 7 ≥ €1,000, which is the size test. The size filter could never remove anything the amount
filter had not already removed. It was deleted; one parameter is easier to explain and impossible to
set inconsistently.

### 7.2 Why €1,000

Two things the data said about the same number.

**Below €1,000/week, silence is normal.** Share of providers with at least one full zero-traffic day in
four normal weeks (20 Aug – 16 Sep), by size:

| Normal week wager | Lines | ≥1 zero day | Mean zero days / 28 |
| --- | --- | --- | --- |
| < €100 | 276 | 99.6% | 22.5 |
| €100–500 | 157 | 87.9% | 10.9 |
| €500–1,000 | 97 | 68.0% | 7.1 |
| **€1,000–2,000** | 113 | **42.5%** | 4.2 |
| €2,000–5,000 | 160 | 35.0% | 2.4 |
| €5,000–10,000 | 140 | 37.1% | 2.0 |
| €10,000–50,000 | 314 | 15.9% | 1.1 |
| €50,000+ | 792 | 2–9% | < 1 |

Below €1,000 silence is the majority behavior; above it, the minority, falling with size. The 552 lines
under €1,000 are 27% of lines and 0.001% of wager.

**Below €1,000, a gap is within the ordinary swing.** Standard deviation of each provider's four normal
weekly totals, median by size:

| Normal week wager | Weekly SD | as % of week |
| --- | --- | --- |
| €1,000–2,000 | €913 | 62% |
| €2,000–5,000 | €1,644 | 52% |
| €5,000–10,000 | €3,770 | 56% |
| €10,000–50,000 | €8,851 | 41% |
| €1M+ | €997,122 | 18% |

€1,000 clears one SD for the smallest providers that can appear on the table. Because a provider can
show at most seven days of its own normal, the same floor is strict for small providers (dark most of
the week) and sensitive for large ones (one day is enough).

**Alternatives.** €500 (the first guess) is below the natural swing of every provider on the table.
€2,000 was used for a while as the amount threshold; it was fine on the data but harder to defend as a
single number once the size test disappeared, and it hid a handful of €2,000–3,000/week casinos going
dark for most of a week. €10,000 (where silence becomes exceptional, 16%) was rejected: it would drop
casinos whose €5,000-a-week providers went dark for 2–3 days — small, real, and the kind of case a KAM
wants.

### 7.3 One threshold for everyone?

Checked by provider and by casino size. Every provider shows the same shape (≤€1k: 86–100% with zero
days; €1k–10k: 31–62%; ≥€10k: 0–13%). Among €1k–10k providers, the zero-day rate is 35–50% whether the
casino is tiny or huge. Silence is a property of the provider's size, not of the casino or the brand, so
one number is justified.

### 7.4 What the threshold costs

Production week, casinos / cells on the page:

| Setting | Casinos | Cells |
| --- | --- | --- |
| none | 252 | 509 |
| €1,000 | 107 | 174 |
| €2,000 | 83 | 120 |
| €10,000 | 49 | 66 |

Synthetic dataset (this repository): none → 63 / 167; **€1,000 → 30 / 53**.

Any provider at €10,000/week or more (99.98% of wager) appears after a single zero day. What €1,000
can hide is a very small casino losing under €1,000 in a week.

### 7.5 Honesty about the sample

Four weeks is one window. The cross-section is large (2,049 lines), so the *shape* is robust; the
*time coverage* is thin, so seasonality is untested. The guide says "in the weeks we measured", the
analysis is re-run quarterly, and the number is a parameter.

## 8. Report day

`MAX(date)` in the source, not `TODAY()`. The pipeline lands a day's data some hours after midnight;
anchoring to the clock would make every morning's view describe a window whose last day is empty.
Verified on production that the last day is complete before the weekly send; if the refresh ever moves
earlier, this must become the last *full* day or every provider gains a false zero day.

## 9. Email, not dashboard

The alert goes out as a Tableau subscription image. That rules out legends that need hovering,
filters, and anything that only makes sense interactively. It is why the header states the windows and
the rule in one sentence, why color is a single ramp that reads without a legend, and why the
analyst-level detail (baseline days, average, zero days) lives in the tooltip on the server version and
nowhere else.
