# Design decisions

Each section is one choice on the shipped page (`1. Final`): what it does, and why. Things I tried and dropped are listed once at the end. Figures come from four weeks of production data (anonymized) and, where noted, from the synthetic dataset in this repository.

---

## 1. Two actions in one page

1. **Clean up** — casinos with no traffic for 50 days: close the line, stop maintaining it.
2. **Call** — casinos where a provider went silent this week: find out why (lobby position,
   promotion, integration, switched off) before more of the week's wager is gone.

That is the whole page. A casino down 30% that never hit zero, or a provider doing a few euros a week, is not one of these two actions and is not on the page.

## 2. Less noise, without hiding things

- **One threshold, printed on the page.** 
  A provider shows only if its silent days cost at least €1,000. Rows are removed by a stated number, not by someone's view of what matters. Lines it
  hides still reach the 50-day list if they stay silent.
- **Everything that passes is ranked.** 
  One number per casino (Potential Wager), sorted. Stop reading whenever you like; the most important rows came first.

## 3. Row = casino, cell = provider, two tables

A KAM acts per casino — one call — so a row is a casino; the provider cells are what to say on
that call. The second table (*No traffic 50 days*) is the cleanup list, with go-live dates so "long
dead" can be told from "not started yet".

## 4. Color = how long; blank = nothing to do right now

One 7-step hue for one variable: days without traffic this week. 
Every reason for "no action": ran every day, not set up here, under the threshold, no baseline.

## 5. "No traffic entire casino" is its own column

A row can be fully colored because each provider had its own silent days, without the casino ever being dark as a whole. That is a different fact, so it gets its own number instead of being guessed from the colors.

## 6. Potential Wager = rate × silent days

> average daily wager **on days with traffic** in the 43 days before this week
> × zero-traffic days this week, per provider, summed per casino

- Only days with traffic count in the average, so a casino that stopped two weeks ago keeps its real rate, and a casino live for three days is measured on those three days.
- The flip side: a line that ran on 2 days out of 43 is measured on those 2. This is shown in the tooltip (days the average rests on) rather than filtered out, because a minimum-days guard would also hide newly live casinos.
- The same condition (zero days ≥ 1 and potential ≥ threshold) drives the entry filter, the cell number, the color and the row total, so the row total always equals the sum of its cells.

## 7. One threshold: potential wager ≥ €1,000

**Why one.** A size test ("provider usually ≥ €1,000/week") is implied by the amount test (potential ≥ €1,000 ⇒ avg daily × 7 ≥ €1,000), so it was dropped.

**Why €1,000.** Two things the data says:

*Below €1,000/week, silence is normal.* Share of providers with ≥ 1 full zero-traffic day in four normal weeks:

| Normal week wager | Lines | ≥1 zero day |
| --- | --- | --- |
| < €100 | 276 | 99.6% |
| €100–500 | 157 | 87.9% |
| €500–1,000 | 97 | 68.0% |
| **€1,000–2,000** | 113 | **42.5%** |
| €2,000–5,000 | 160 | 35.0% |
| €5,000–10,000 | 140 | 37.1% |
| €10,000–50,000 | 314 | 15.9% |
| €50,000+ | 792 | 2–9% |

Lines under €1,000 are 27% of lines and 0.001% of wager.

*Below €1,000, a gap is within the ordinary swing.* 
Median week-to-week SD of a provider's wager:
€913 for €1,000–2,000/week lines, €1,644 for €2,000–5,000, €3,770 for €5,000–10,000. €1,000 clears one SD for the smallest lines that can appear; a large line passes after a single silent day.

**Checked across groups.** The same shape holds for every provider and for tiny and huge casinos alike, so one number serves everyone.

**What it costs.** 
Production week: 252 casinos / 509 cells with no threshold → 107 / 174 at €1,000. 
Synthetic data: 63 / 167 → 30 / 53. Providers at €10,000+/week (99.98% of wager) appear after one silent day; what can be hidden is a very small casino losing under €1,000 in a week.

**The sample.** Four weeks, 2,049 lines: the shape is solid, seasonality is untested. The number is a parameter and is re-checked quarterly.

## 8. Report day

`MAX(date)` in the source, not `TODAY()`, so an empty last day never shows up as a wave of false alerts. 
This week = 7 days ending there; baseline = the 43 days before. Dates are printed on the page.

## 9. Built for email

The alert is a weekly subscription image, sent only when the list is not empty. Nothing on it needs a hover or a click
Detail(baseline days, average, zero days) lives only in the tooltip on the interactive version in case anyone wants more information.

---

## Appendix — tried and dropped

| Tried | Dropped because |
| --- | --- |
| Filtering out rows by judgment ("nobody acts on these") | Team refused: every row had a reason to exist. Replaced by one printed rule plus ranking. |
| Four routed tables: never live / underperforming / casino dark / provider stopped (`2. Rejected`) | "Too much stuff!!" answered *what kinds of silence exist?*, not *what to do next?* |
| Four-color legend (stopped / down / normal / never used) | Nobody could say what each color meant. |
| Also flagging providers 30% below normal that still ran every day | Doubled the legend, lit up the dominant provider on nearly every row. Separate page. |
| Baseline = 50-day average including silent days (original) | Diluted by the casino's own silence. |
| Potential = normal week (4 weeks) − this week | Matched rate × days on median (ratio 1.00), but a stopped casino's number shrank every week. |
| Minimum 14 baseline days with traffic | Would hide newly live casinos. Shown in tooltip instead. |
| Two thresholds (size and amount) | Size test is implied by the amount test. |
