# Austrian Law Watcher 🇦🇺⚖️

A [Hermes agent skill](https://hermes-agent.nousresearch.com/docs) that monitors new Austrian government bills (**Ministerialentwürfe**) on the official Federal Parliament of Austria page, lists all new ones from the last N days (default: 8), briefly summarizes each from its official **"Kurzinformation"**, and provides **direct links** to every bill's detail page as well as to the overview list.

> Author: Felix [Morocutti](https://github.com/FelixM2011) · License: [MIT](LICENSE)

## What it does

- Surfs `https://www.parlament.gv.at/recherchieren/gegenstaende/ministerialentwuerfe`
- Parses the results table (`Einlangen`, `Betreff`, `Nr./ME`, `Ressort`, `Frist`, `Stellungnahmen`)
- Filters bills whose *Einlangen* date is inside the lookback window (default: last 8 days, both ends inclusive)
- For every hit: open the detail page and summarize from the ministry's own „Ziele"/„Inhalt" bullets — nothing invented
- Delivers a compact digest with one section per bill + the overview link at the end

See [`references/sample-digest.md`](references/sample-digest.md) for a real sample output.

## Install (Hermes)

```bash
git clone https://github.com/FelixM2011/AustrianLawWatcher.git ~/.hermes/skills/austrian-law-watcher
```

Required toolsets: `web`, `browser`. Tagged in the skill frontmatter, so Hermes picks it up directly.

## Usage

- Interactively — ask e.g.: *„Welche neuen österreichischen Gesetzesentwürfe gibt es?"* or *"Liste Ministerialentwürfe der letzten 8 Tage mit kurzen Zusammenfassungen"*
- As a scheduled cron job: attach this skill to a daily job; when nothing new comes up it answers `[SILENT]` and the scheduler suppresses delivery automatically.

## Output format (excerpt)

```markdown
# Neue Ministerialentwürfe – letzte 8 Tage (ab 03.10.2026 bis 11.10.2026)

Auf der Übersichtsseite sind in den letzten 8 Tagen **X** neue Ministerialentwürfe gelistet:

## 1. <Betreff> – <Nr./ME>
- **Ressort:** … · **Einlangen:** DD.MM.YYYY · **Frist:** DD.MM.YYYY
- **Zusammenfassung:** 1–3 Sätze aus der Kurzinformation
- **Link:** https://www.parlament.gv.at/gegenstand/XXVIII/ME/<NUM>

…(je Entwurf ein Abschnitt, neueste zuerst)
```

## Notes & limitations

- The overview table is rendered client-side (JavaScript); the skill uses a headless browser with a short wait before parsing — plain HTTP scraping of that page is unreliable.
- Default filter on the site shows only the current legislative period; longer lookbacks require adjusting that filter first.
- Summaries are grounded in the official "Kurzinformation" block on each bill's detail page and written in German, matching the source data.

## Repository layout

```
├── SKILL.md                 # the skill itself (frontmatter + procedure)
├── references/
│   └── sample-digest.md     # real example output from a live run
├── LICENSE                  # MIT © 2026 Felix Morocutti
└── README.md                # this file
```
