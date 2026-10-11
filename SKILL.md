---
name: austrian-law-watcher
description: Watch new Austrian government bills (Ministerialentwürfe) on parlament.gv.at, list the latest ones with summaries and direct links.
version: 1.0.0
author: Felix Morocutti https://github.com/FelixM2011
license: MIT
metadata:
  hermes:
    tags: [austria, legislation, monitoring, government]
    requires_toolsets: [web, browser]
---

# Austrian Law Watcher (Ministerialentwürfe)

Monitors the official Federal Parliament of Austria page for **Ministerialentwürfe** (government bills in pre-parliamentary review). Lists every new bill submitted within a lookback window (default: last 8 days), briefly summarizes each one, and provides direct links to both the individual bill detail pages and the overview list.

## Source

- Overview page: `https://www.parlament.gv.at/recherchieren/gegenstaende/ministerialentwuerfe`
- Bill detail page pattern: `https://www.parlament.gv.at/gegenstand/<PERIOD>/ME/<NUM>` (e.g. `.../XXVIII/ME/143`)

## When to Use

- A cron job using this skill fires ("Neue Gesetzesentwürfe")
- User asks what new Austrian government bills are in progress right now
- User wants a digest of bill submissions over the last N days

Columns on the overview table:

| Column | Meaning |
|---|---|
| **Einlangen** | Date the bill was submitted/published on the page — THIS is the filter column |
| **Betreff** | Title; clickable link to the bill detail page |
| **Nr.** | Bill number, format `<number>/ME` (e.g. `143/ME`) |
| **Ressort** | Responsible ministry (abbreviation + full name) |
| **Frist** | End of the pre-parliamentary review period (Begutachtungsfrist) |
| **Stellungnahmen** | Number of third-party opinions submitted so far |

## Procedure

1. **Compute the date window**: today minus N days (default **N = 8**), in local `DD.MM.YYYY` format. The list is sorted by *Einlangen* descending by default, so page 1 almost always covers an 8-day window — paginate further only if rows on the current page are still inside the window and more may follow.

2. **Open the overview page** with `browser_navigate`:
   `https://www.parlament.gv.at/recherchieren/gegenstaende/ministerialentwuerfe`
   - Wait ~5 s before snapshotting: the result table is rendered dynamically (a "loading" placeholder shows until data arrives). Snapshotting immediately yields an empty main area.
   - A cookie-consent dialog can appear ("Alle Akzeptieren") — accept it once so the list renders. If a previous snapshot already contains the loaded table, parse that content instead of re-navigating.

3. **Parse the table** (from the browser snapshot, or `browser_console` over the DOM for robustness). Per row read:
   - *Einlangen* date (filter column)
   - *Betreff* text + its link URL (detail page, relative like `/gegenstand/XXVIII/ME/143`)
   - *Nr.* (e.g. `143/ME`)
   - *Ressort*, *Frist*, *Stellungnahmen* count

4. **Filter** rows whose *Einlangen* date falls inside the window (both ends inclusive). The "zuletzt aktualisiert am" stamp above the table is context only, not a filter source.

5. **Summarize each bill**: open its detail page (`browser_navigate` — loads fast; `web_extract` also fine) and use the **"Kurzinformation"** block: it holds official „Ziele" (goals) and „Inhalt" (content) bullets from the responsible ministry. Condense into 1–3 sentences, adding nothing not visible on the page. Fallback if no Kurzinformation exists: one grounded sentence based on the *Betreff* line plus ministry + procedural status ("im Begutachtungsverfahren"). Deep dives into the **Dokumente** (PDFs of Gesetzestext, Vorblatt/WFA, Erläuterungen) only on explicit user request — keep default runs light. If two bills are clearly a package (same day, same theme), note that in one closing line.

6. **Format output** — compact digest:

   ```
   # Neue Ministerialentwürfe – letzte N Tage (ab DD.MM.YYYY bis DD.MM.YYYY)

   Auf der [Übersichtsseite](https://www.parlament.gv.at/recherchieren/gegenstaende/ministerialentwuerfe) sind in den letzten 8 Tagen **X** neue Ministerialentwürfe gelistet:

   ## 1. <Betreff> – <Nr./ME>
   - **Ressort:** <Ministerium> · **Einlangen:** DD.MM.YYYY · **Frist:** DD.MM.YYYY
   - **Zusammenfassung:** <1–3 sentences>
   - **Link:** https://www.parlament.gv.at/gegenstand/<PERIOD>/ME/<NUM>

   … (one section per bill, newest first)

   ---
   Übersichtsseite: [parlament.gv.at – Ministerialentwürfe](https://www.parlament.gv.at/recherchieren/gegenstaende/ministerialentwuerfe)
   ```

7. **Delivery**: designed for scheduled cron runs — the final response IS the digest (the scheduler delivers it). If genuinely nothing new was found in window, respond exactly `[SILENT]` to suppress delivery; never pad with invented items. When asked interactively instead of silently: state "0 neue Entwürfe im Zeitraum" and show the two oldest rows as a sanity check.

## Pitfalls

- The table is JS-rendered — `web_extract` alone tends to return loading stubs or missing tables; use the browser, wait ~5 s, then parse
- Pagination defaults: 10 rows per page (~140+ bills total for the current Gesetzgebungsperiode); the default filter shows only the *current* period (e.g. XXVIII starting Oct 2024) — switch/reset that filter before expecting older entries
- Row links are relative — prefix `https://www.parlament.gv.at` when printing detail URLs; never reconstruct a number from memory if it is not in the row
- *Einlangen* dates cluster: several bills can carry the same stamp (batch submission by one ministry) — still separate list items, same day noted once
- Keep summary language German to match source data

## Verification

- Every listed bill's direct link was actually present in the parsed table row / detail page
- Every *Einlangen* date sits inside the reported window, and dates shown were read from rows (not estimated)
- Overview URL appears exactly once at the end, verbatim: `https://www.parlament.gv.at/recherchieren/gegenstaende/ministerialentwuerfe`
- If loading fails or no table row is visible → report that error honestly. Never fabricate bills, dates, numbers
