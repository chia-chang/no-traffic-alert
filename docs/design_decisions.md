# Design decisions

Each section is one choice on the shipped page (`1. Final`): what the page does, and why. Things I tried and dropped are also included at the end, so the main text describes the page as it is. Figures come from four weeks of production data (anonymized) and, where noted, from the synthetic dataset in this repository.

---

## 1. Two actions in one page

The page exists for two things a KAM can actually do:

1. **Clean up** — casinos with no traffic at all for 50 days: close the line and stop maintaining settings for it.
2. **Call** — casinos where at least a provider went silent this week: find out why (lobby position, promotion, integration problem, switched off) before more of the week's wager is gone.

That is the whole page: one table for this week, one cleanup list for 50 days. 
A casino that is own 30% but never hit zero, or a provider doing a few euros a week, is not one of these two actions and is not on the page. Earlier versions tried to cover those too. See the appendix for more details.

## 2. Making content more digestable by not bombing them with too much information

- **Adding potential wager < €1,000.** A provider appears only if its silent days cost at least €1,000. That does remove rows — but by a single number that is printed in the header, backed by data, and adjustable as a parameter, not by someone's opinion of what matters. Lines it hides are not gone for good: if they stay silent they surface on the 50-day list.
- **Everything that passes is ranked.** Each casino gets one number (Potential Wager), the table is sorted by it, and a reader can stop at any point having seen the most important rows first.

## 3. Row = casino, cell = provider, two tables

A KAM acts around casinos, so a row is a casino, and the provider grid inside the row is what to say on that call. 
The second table (*No traffic 50 days*) is the same question stretched to the whole window: lines nobody uses any more, with go-live dates so "long dead" can be told from "not started yet".

## 4. Color tells how long of a problem this is, and blank means nothing act on at the moment

Cell color is a single 7-step hue for one variable: days without traffic this week. 
Every reason for blank: ran every day, not set up here, under the threshold, no baseline.

## 5. "No traffic entire casino" is its own column

A row can be fully colored because each provider had its own silent days without the casino ever being dark as a whole. Whether *all* providers were silent on the same day is a different fact, so it is computed at casino level and shown as a number rather than left for the reader to infer from the colors.

## 6. Potential Wager = rate × silent days

For each provider with at least one zero-traffic day this week:

> average daily wager **over the days it had traffic** in the 43 days before this week
> × zero-traffic days this week

summed per casino. It answers "at the rate this provider normally runs, what did the silent days
cost?", and it is bounded: at most seven days of the provider's own normal.

**Why the baseline only counts days with traffic.** A casino that stopped two weeks ago should still
be measured at the rate it ran at, not at an average diluted by its own silence — otherwise its
number shrinks every week, exactly when it should stay loud. A casino live for three days should be
measured on those three days, not on a 43-day average that is mostly zeros.

**The cost, disclosed rather than filtered.** A provider that ran on two days out of 43 is measured
on those two, and can show a number it never actually made in a week. A minimum-days guard would fix
that but would also hide newly live casinos — the ones most likely to have an integration problem.
So the tooltip shows how many days the average rests on, and the guide says so.

**Why the row total equals the sum of its cells.** The same condition (zero days ≥ 1 and gap ≥
threshold) is used in the entry filter, the cell number, the cell color and the row total. See
`calculated-fields.md` §6.

## 7. One threshold: potential ≥ €1,000

### 7.1 Why one parameter

A size test ("provider usually does ≥ €1,000/week") would be redundant: potential ≥ €1,000 already
implies average daily × 7 ≥ €1,000. One parameter is easier to explain and cannot be set
inconsistently.

### 7.2 Why €1,000

Two things the data says about the same number.

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

Below €1,000 silence is the majority behavior; above it, the minority, falling with size. The 552
lines under €1,000 are 27% of lines and 0.001% of wager.

**Below €1,000, a gap is within the ordinary swing.** Standard deviation of each provider's four
normal weekly totals, median by size:

| Normal week wager | Weekly SD | as % of week |
| --- | --- | --- |
| €1,000–2,000 | €913 | 62% |
| €2,000–5,000 | €1,644 | 52% |
| €5,000–10,000 | €3,770 | 56% |
| €10,000–50,000 | €8,851 | 41% |
| €1M+ | €997,122 | 18% |

€1,000 clears one SD for the smallest providers that can appear on the table. Because a provider can
show at most seven days of its own normal, the same floor is strict for small providers (dark most of
the week before they show) and sensitive for large ones (one day is enough).

### 7.3 One number for everyone

Checked by provider and by casino size: every provider shows the same shape (≤€1k: 86–100% with
zero days; €1k–10k: 31–62%; ≥€10k: 0–13%), and among €1k–10k providers the zero-day rate is 35–50%
whether the casino is tiny or huge. Silence is a property of the provider's size, not of the casino
or the brand.

### 7.4 What it costs

| Threshold | Casinos | Cells |
| --- | --- | --- |
| none | 252 | 509 |
| **€1,000** | **107** | **174** |
| €2,000 | 83 | 120 |
| €10,000 | 49 | 66 |

(Production week. On the synthetic dataset: none → 63 / 167; €1,000 → 30 / 53.)

Any provider at €10,000/week or more — 99.98% of wager — appears after a single zero day. What
€1,000 can hide is a very small casino losing under €1,000 in a week.

### 7.5 The sample

Four weeks is one window. The cross-section is large (2,049 lines), so the shape is robust; the time
coverage is thin, so seasonality is untested. The guide says "in the weeks we measured", the analysis
is re-run quarterly, and the number is a parameter so it can move without touching a formula.

## 8. Report day and windows

Report day = `MAX(date)` in the source, not `TODAY()`: data lands hours after midnight, and a
clock-anchored window would show an empty last day as a wave of false alerts every morning. This
week = 7 days ending on the report day; baseline = the 43 days before that (the source holds 50).
The dates are printed on the page. If the refresh ever moves earlier than the last full day, the
report day must become the last *complete* day or every provider gains a false zero day.

## 9. Built for email

The alert is a weekly Tableau subscription image, sent only when the list is not empty. That rules
out anything that needs hovering or clicking: the header states the window and the rule in one
sentence, the color ramp reads without a legend, and analyst-level detail (baseline days, average,
zero days) lives in the tooltip on the interactive version and nowhere else.

---

## Appendix — tried and dropped

| Tried | Dropped because |
| --- | --- |
| Filtering out rows by judgment ("nobody acts on these") | Team refused: every row had a reason to exist. Replaced by one printed rule plus ranking (§2). |
| Four routed tables — never live / underperforming (under €500 in 50 days) / casino dark / provider stopped (`2. Rejected`) | "Too much stuff"; answered the analyst's question (*what kinds of silence exist?*), not the KAM's (*who do I call?*). |
| Four-color legend (red stopped · yellow down · green normal · gray never used) | Reviewers could not say what each color meant. |
| Also flagging providers 30% below normal that still ran every day (a step between `2. Rejected` and `1. Final`) | Doubled the legend, lit up the dominant provider on nearly every row. A traffic-drop alert is a separate page. |
| Baseline = 50-day average including silent days (original) | Diluted by the casino's own silence. |
| Baseline = 4 full weeks; potential = normal week − this week (actual gap) | Verifiable against client numbers and matched rate × days on median (ratio 1.00), but a stopped casino's number shrank every week. Business asked for rate × days. |
| Minimum 14 baseline days with traffic | Would hide newly live casinos. Disclosed in tooltip instead. |
| Two thresholds (size ≥ €1,000/week and amount ≥ €1,000) | Size test is implied by the amount test. |
| Amount threshold €500 | Below the natural swing of every provider on the table — filters nothing. |
| Amount threshold €2,000 | Fine on the data, but hid €2,000–3,000/week casinos dark most of a week. |
| Threshold €10,000 | Drops €5,000/week providers dark 2–3 days — small, real, the case a KAM wants. |
