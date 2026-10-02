# Calculated fields

Every calculated field in the workbook, in dependency order, with what it is for and why it is
written that way. Formulas are exactly as they appear in the packaged workbook.

Throughout, the grain of the source table is **environment × casino × provider × day**, with an
additional set of casino-level rows distinguished by `Grouping Set Level`.

---

## 1. The window

Everything downstream hangs off one field, and it is data-driven rather than clock-driven.

```
Report Day     =  { MAX([Date Only]) }
Date Start _ 7 =  DATE(DATEADD('day', -6, [Report Day]))
Date Start _ 50 = DATE(DATEADD('day', -49, [Report Day]))
```

**Why `MAX(DateOnly)` and not `TODAY()`.** The pipeline lands a day's data some hours after
midnight. Anchoring to the clock means that every morning, for a few hours, the report silently
describes a window whose last day is empty — and shows it as a wave of new zero-traffic alerts.
Anchoring to the data means the window only advances once the data exists.

The cost is that the window moves whenever the source refreshes, which makes side-by-side
verification harder: two sheets compared across a refresh are not comparing the same period. All
three date fields are printed in the page header so a reader can always see which window they are
looking at.

```
Date Filter _ 7 days = [Date Only] >= [Date Start _ 7]
50 days filter       = [Date Only] > DATEADD('day', -50, [Report Day])
```

> ⚠️ **Never add `Date Filter _ 7 days` to context.** FIXED LODs are evaluated *before* dimension
> filters but *after* context filters. Every 50-day baseline in this workbook — active rate,
> average daily wager, days with traffic — depends on that ordering. Promoting the 7-day filter to
> a context filter silently collapses all of them to 7 days. Nothing errors; the numbers just
> quietly become wrong.

---

## 2. Turning a flag into a date

```
Date _ Have Traffic = IF [Traffic Flag] = 'Have Traffic' THEN [Date Only] END
Date _ No Traffic   = IF [Traffic Flag] = 'No Traffic'   THEN [Date Only] END
```

**Why convert a flag to a nullable date.** It makes `COUNTD` and `MAX` do the counting. `COUNTD`
of a nullable date gives "how many distinct days had traffic" with no conditional aggregation, and
`MAX` gives "the last day it had traffic" for free. It also means a combination that has *never*
had traffic yields `NULL` rather than `0`, which is the distinction tables 1 and 4 are built on —
never-live and stopped-yesterday are different questions and should not both be zero.

---

## 3. Casino level

```
Casino Last Traffic Day = { FIXED [Environment code],[Casino ID] : MAX([Date _ Have Traffic]) }
Casino no traffic filter = ISNULL([Casino Last Traffic Day])
Days since last traffic  = DATEDIFF('day', [Casino Last Traffic Day], [Report Day])

Zero Traffic Days = 7 - { FIXED [Environment code],[Casino ID] :
                          COUNTD(IF [Date Only] >= [Date Start _ 7]
                                 THEN [Date _ Have Traffic] END) }

Days with traffic (50d) = { FIXED [Environment code],[Casino ID] :
                            COUNTD(IF [Date Only] >= [Date Start _ 50]
                                   THEN [Date _ Have Traffic] END) }

Total wager (50d) = { FIXED [Environment code],[Casino ID] :
                      SUM(IF [Date Only] >= [Date Start _ 50] THEN [Wager Eur] END) }

Casino Go Live Date = { FIXED [Environment code],[Casino ID] :
                        MIN(IFNULL([Go Live Date], DATE('2099-12-31'))) }
```

**Why the date restriction lives inside the LOD instead of on the sheet.** Because these have to
be 50-day baselines regardless of what the sheet is filtered to. A sheet showing 7 days still needs
the 50-day average to compare against. Putting the window inside the expression makes the field
mean the same thing on every sheet it appears on.

**Why `IFNULL(..., 2099-12-31)` in the go-live date.** `MIN` skips nulls, so a casino with one
provider missing a go-live date would silently report the *other* provider's date as the casino's.
Substituting a sentinel far-future date makes a missing value lose the `MIN` comparison instead of
disappearing from it — a null then surfaces as an obviously wrong date rather than a plausible
wrong one.

---

## 4. Provider level

```
Provider last traffic day = { FIXED [Environment code],[Casino ID],[Game Provider] :
                              MAX([Date _ Have Traffic]) }

Provider have-traffic days = { FIXED [Environment code],[Casino ID],[Game Provider] :
                               COUNTD(IF [Date Only] >= [Date Start _ 50]
                                      THEN [Date _ Have Traffic] END) }

Provider days dark = DATEDIFF('day', [Provider last traffic day], [Report Day])

Casino Provider ZERO traffic days =
  7 - { FIXED [Environment code],[Casino ID],[Game Provider] :
        COUNTD(IF [Date Only] >= [Date Start _ 7] THEN [Date _ Have Traffic] END) }
```

`Provider have-traffic days = 0` is the workbook's definition of **"never carried traffic in the
window"**, and it is used everywhere that distinction matters. It is deliberately a *condition*
rather than a list of provider names: one provider had been enabled in bulk across most of the
client base and never went live, and hard-coding its name would have been a rule that broke
silently the next time somebody did the same thing with a different provider.

⚠️ **Aggregating a FIXED LOD on a shelf.** These are constant within their own grain, so putting one on Rows with the default `SUM` multiplies it by the number of underlying records — a days-since value of 30 renders as 11,370. Use `MIN`, `MAX` or `ATTR`. The symptom is a number that is suspiciously close to a round multiple of the true one.

---

## 5. Routing

The four tables are mutually exclusive, and the exclusivity is enforced here rather than by
convention.

```
Casino Provider Filter = { FIXED [Environment code],[Casino ID] : MAX(
    7 - { FIXED [Environment code],[Casino ID],[Game Provider] :
          COUNTD(IF [Date Only] >= [Date Start _ 7] THEN [Date _ Have Traffic] END) } ) }

Casino has stopped provider = { FIXED [Environment code],[Casino ID] :
    MAX(IF [Provider have-traffic days] > 0
           AND [Casino Provider ZERO traffic days] >= 1
        THEN 1 ELSE 0 END) } = 1

less than threshold filter = [Total wager (50d)] < [Cleanup threshold]      // parameter
```

`Casino Provider Filter` is a **nested FIXED LOD**: the inner expression computes zero-days per
provider, the outer takes the worst one per casino. This is what lets a casino-level table filter
on a provider-level condition without changing the grain of the view.

`Casino has stopped provider` is the same idea with the never-live providers excluded, and it is
the correct routing test for table 4 — see the note at the end.

Routing logic:

| Table | Condition |
|---|---|
| ① Never live | `Casino no traffic filter = TRUE` |
| ② Underperforming | not ①, and `less than threshold filter = TRUE` |
| ③ Casino dark this week | not ①, not ②, and `Zero Traffic Days >= 1` |
| ④ Provider gap | not ①, not ②, `Zero Traffic Days = 0`, and `Casino has stopped provider` |

Because ③ requires at least one casino-wide zero day and ④ requires none, the two cannot both
match. ① and ② are subtractive. No casino can appear twice.

---

## 6. Ranking

```
Potential Wager = { FIXED [Environment code],[Casino ID] :
    SUM(IF [Provider have-traffic days] > 0 AND [Date Only] >= [Date Start _ 50]
        THEN [Wager Eur] / [Provider have-traffic days] * [Casino Provider ZERO traffic days]
        END) }

Casino €/day when running = { FIXED [Environment code],[Casino ID] :
    SUM(IF NOT ISNULL([Provider last traffic day]) AND [Date Only] >= [Date Start _ 50]
        THEN [Wager Eur] / [Provider have-traffic days] END) }
```

`Wager / have-traffic days` is the provider's average daily wager **on the days it was actually
running** — dead days do not dilute it, so it measures the size of the line rather than its recent
uptime. Multiplying by this week's zero-traffic days estimates what that stoppage cost, and summing
across providers gives a per-casino figure the table can sort on.

**Why this is bounded to the week rather than cumulative.** `Casino Provider ZERO traffic days` can
only be 0–7, so the multiplier is bounded by construction. An unbounded version — average daily
wager × days-since-last-traffic — ranks a small line down for three weeks above a large line that
broke yesterday, which inverts the order you actually want: the fresh break is the recoverable one.
The bound is what keeps the list sorted by actionability instead of by accumulated regret.

The `Provider have-traffic days > 0` guard is load-bearing. Without it, a provider that never
carried traffic has a zero denominator and contributes either nothing or an error depending on the
path — and a never-live line should not be able to claim it "missed" revenue it never earned.

---

## 7. Display states

```
Cell status = IF [Provider have-traffic days] = 0 THEN 'Never used (50d)'
              ELSE STR([Casino Provider ZERO traffic days]) END

Day status  = IF [Provider have-traffic days] = 0 THEN 'Never used (50d)'
              ELSE [Traffic Flag] END

Cell label  = IF [Provider have-traffic days] = 0 THEN '–'
              ELSEIF [Casino Provider ZERO traffic days] = 0 THEN ''
              ELSE STR([Casino Provider ZERO traffic days]) END

Never-used providers = { FIXED [Environment code],[Casino ID] :
                         COUNTD(IF [Provider have-traffic days] = 0 THEN [Game Provider] END) }
```

**Why a string rather than a colour rule.** The colour encoding is a discrete field, so returning
`'Never used (50d)'` as a *member* lets it take its own grey swatch alongside the 0–7 gradient,
and the legend then carries the explanation in plain words — which matters because the legend is
part of the emailed image. A continuous colour field would have rendered these as nulls, which is
both untypable and unlabelled.

**Why grey rather than hidden.** Removing them entirely would hide a genuinely broken line if that
provider ever did go live, and it would flatten the diagnostic contrast that makes a row readable:
four green cells and one red says "provider problem"; five red says "the casino stopped". Greying
keeps both the information and the contrast while removing the visual urgency — the loudest thing
on the page should not be the most meaningless.

---

## 8. Access control

```
Assigned to KAM = TRUE        // demo build
Team scope      = TRUE        // demo build
```

In the internal build these are row-level security predicates comparing the signed-in user against
the account owner, applied as **context filters** so each manager sees only their own portfolio.
They are stubbed to `TRUE` in this public version, both because there is no signed-in user on
Tableau Public and because the original expressions contained real account identifiers.

Note the interaction with section 1: these are context filters, so they *do* run before the FIXED
LODs. That is intentional and correct — a manager's baselines should be computed within their own
portfolio. It is also the reason the ordering rule matters enough to be worth stating twice.
