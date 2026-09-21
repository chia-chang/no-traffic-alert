# Zero Traffic Alert — how to read it

Sent out once a week. If nothing needs attention, you get no email.

> **Note:** This is the public version of the guide, written against a synthetic dataset.
> Provider names (`Provider 1 (Live)`, `Provider 2` …) and all figures are generated.

---

## Definitions

- **50d** — the 50-day window ending on the report day. Both dates are printed at the top of the page.
- **This week** — the last 7 days ending on the report day.
- **Total wager (50d)** — sum of wager across the 50-day window, all providers.
- **Potential Wager** — for each provider that had no traffic on some days this week: its average
  daily wager (over the days it *did* have traffic in the 50d window) × the number of days it had
  no traffic this week. Summed per casino.
  **It is an estimate for ordering, not a financial figure.**

  Worked example (illustrative numbers, not from any casino in the dataset):

  | Provider | Total wager (50d) | Days with traffic (50d) | Average daily wager | Zero traffic days | Potential wager |
  |---|---|---|---|---|---|
  | Provider 1 (Live) | 15,000,000 | 30 | 500,000 | 1 | **500,000** |
  | Provider 1 (RNG) | 50,000 | 50 | 1,000 | 0 | 0 |
  | Provider 2 | 1,500,000 | 20 | 75,000 | 4 | **300,000** |
  | Provider 3 | — | 0 | — | 7 | 0 |

  → Potential Wager = 500,000 + 300,000 = **800,000**

  Note the last row: a provider that has never carried traffic contributes nothing, however many
  zero days it has. That is deliberate — see *Good to know* below.

---

## 1 · Never live in the window (red)

**Casinos with zero traffic across the whole 50-day window**, or since their go-live date if they
went live more recently.

A ● means that provider is live for this casino.

| Column | Meaning |
|---|---|
| Casino Go Live Date | When this casino went live. An old date and still no traffic = long dead. A recent date = possibly just not started yet. |

→ **Does this line still need to exist?** If not, close it so we stop maintaining settings for it.

---

## 2 · Underperforming casinos (brown)

**Casinos that had some traffic, but less than the cleanup threshold in total wager across 50 days.**
The threshold is shown in the page header and can be adjusted.

A ● means that provider is live for this casino.

| Column | Meaning |
|---|---|
| Days since last traffic | How many days ago it last had any traffic. |
| Days with traffic (50d) | Of the last 50 days, how many had any traffic at all. |
| Total wager (50d) | Total wager over the 50 days. |

→ **Same question as table 1: is this line worth keeping?**

---

## 3 · Whole casino went dark this week (orange)

**Running casinos (above the cleanup threshold) that had at least one full day with no traffic
this week.**

● had traffic that day · ✖ no traffic that day · **grey ✖** this provider had no traffic in the
whole 50-day window

| Column | Meaning |
|---|---|
| Casino Last Traffic Day | Last day the casino had any traffic. |
| Days since last traffic | How long ago that was. 0 = it is back today. |
| Zero Traffic Days | How many of the last 7 days the whole casino was dark. |
| Potential Wager | Estimated wager missed this week (see definition above). |

→ **Why did the whole casino stop?** Connection, client-side change, or planned?

---

## 4 · A provider stopped, casino still running (yellow)

**Casinos that ran every day this week, but one or more providers did not.**

| Column | Meaning |
|---|---|
| Potential Wager | Estimated wager missed this week from the stopped providers. |
| Number in each provider cell | How many of the last 7 days that provider had no traffic. Colour runs green (0) → yellow → dark red (7). |
| Grey cell | This provider has had no traffic in the whole 50-day window. |

→ **Is that provider still meant to be live for this client?**

**How to read a row here.** The cells that are *still green* are the useful part. Four green and
one red means the client is alive and reachable and the fault is specific to one integration —
worth a call today. If most of the row is grey, this casino has a lot of enabled-but-unused lines,
which is a provisioning conversation rather than an incident.

---

## Good to know

- **A casino appears in exactly one table.** Tables 1 and 2 are about lines that are barely or
  never used. Tables 3 and 4 are about lines that are normally active and stopped. If you are
  looking for a casino and cannot find it, it is healthy — nothing about it needed reporting.
- **Recently added casinos can land in table 1 or 2 simply because they have not started yet.**
  Check the go-live date before treating it as dead.
- **Grey cells never raise an alert on their own.** A provider that has never carried traffic in
  the window is shown, but it does not pull a casino onto the list and it contributes nothing to
  Potential Wager. This stops one batch of enabled-but-never-live integrations from flooding the
  report — which is what happened in the previous version.
- **Only dates after each casino's go-live date are counted.**

---

![Zero Traffic Alert dashboard](../images/00-header.png)
