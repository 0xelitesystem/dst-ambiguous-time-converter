# DST Ambiguous Time Converter

Paste a column of naive local timestamps and an IANA time zone to find every row that falls in a DST gap (never existed) or fold (happened twice), with its UTC value under the Temporal, Python and PostgreSQL rules, before you migrate the data.

**Live demo:** https://0xelitesystem.github.io/dst-ambiguous-time-converter/

## Use

1. Paste your timestamps into the box, one per line. Or paste a CSV or TSV export with its header row, tick "First line is a header row" and pick the timestamp column. Every other column is carried through to the export untouched.
2. Type the IANA time zone the timestamps were recorded in, for example `America/New_York` or `Europe/Dublin`. Any identifier your browser accepts works, including ones missing from its suggestion list such as `Europe/Kyiv`. The page echoes exactly what you typed, and tells you when your browser resolves it to a different name.
3. If your dates use slashes, choose MM/DD/YYYY or DD/MM/YYYY under Date order. The page never guesses.
4. Press **Analyse column**. You get:
   - a count of rows, normal rows, folds, gaps, empty cells (NaT), rows the strict parser rejected, and DISAGREEMENT rows;
   - one line per gap or fold row with the offset before and after, the gap or fold length, and the UTC value under Temporal 'earlier', 'later', 'compatible' and 'reject', Python fold=0 and fold=1, and PostgreSQL;
   - a prediction of what `Series.dt.tz_localize` in pandas will do with the column, under `ambiguous='raise'` or `'infer'` and `nonexistent='raise'` or `'NaT'`, naming the exact error and row when it will raise;
   - the transitions the page used, with the affected local wall range for each;
   - every rejected or flagged row with the reason (date only, before 1970, an offset with seconds, an offset already present).
5. Pick a policy and copy the full column as CSV (every input column plus `kind`, `status`, `utc`, `offset_used` and an RFC 9557 `ixdtf` string), or copy a pandas `ambiguous=` bool list where True means the earlier instant.

The self test runs on load and shows every assertion. When your browser has Temporal, the page also compares its own classifier with Temporal around every transition from 1970 to 2040 in a spread of zones and with every row you paste, and shows any disagreement in red.

### Use cases

- **Moving a legacy DATETIME column to timestamptz.** Paste the column before the migration and see which rows PostgreSQL will place on the later side of a fold, and by how much that differs from what your Python ETL wrote.
- **Loading a CSV into pandas.** Find out whether `tz_localize` will raise, which row it will name, and whether `ambiguous='infer'` will succeed on your row order, before the job runs at 2 a.m.
- **Auditing a scheduler or meter export.** Exports that store wall clock time lose the offset. The fold rows are the ones where you cannot recover the instant without another column, and this lists them.
- **Picking a policy on purpose.** Compare 'earlier', 'later' and 'compatible' side by side, then export under the one you chose, with an RFC 9557 string such as `2025-11-02T01:30:00-04:00[America/New_York]` that records both the offset and the zone.

## Why this exists

Python and PostgreSQL each follow a published rule, and they disagree by an hour on fold rows. Take `America/New_York 2025-11-02 01:30`:

- Python `datetime` with `zoneinfo` (PEP 495 default fold=0): `2025-11-02T05:30:00Z`.
- Temporal default disambiguation 'compatible': `2025-11-02T01:30:00-04:00`, which is 05:30Z.
- PostgreSQL (`'2025-11-02 01:30'::timestamp AT TIME ZONE 'America/New_York'`): `2025-11-02 06:30:00+00`.
- For the gap row `2025-03-09 02:30`, all three return 07:30Z.

pandas raises by default on both kinds of row. This page shows which rows sit in the disagreement zone and what each side will write. It never labels an offset "DST" or "standard": the tz database encodes Europe/Dublin with a negative save value (`Rule Eire 1996 max - Oct lastSun 1:00u -1:00 -`), so Dublin's summer +01:00 is its standard time in that data. It never assumes a gap or fold is an hour long either: Lord Howe Island moves by 30 minutes (`Rule LH 2008 max - Oct Sun>=1 2:00 0:30 -` on a 10:30 standard offset).

It is one HTML file with no dependencies, no build step, no analytics and no network calls, so the column you paste is never uploaded anywhere. Read the source, host it anywhere, and use it under the MIT licence.

## Privacy

Everything runs in your browser and nothing leaves your machine. There is no server, no analytics and no network request of any kind; the page's Content-Security-Policy blocks them. The only thing stored is your light or dark theme choice, in your browser's local storage.

## Run locally

```
git clone https://github.com/0xelitesystem/dst-ambiguous-time-converter.git
cd dst-ambiguous-time-converter
```

Open `index.html` in any modern browser, or serve the folder:

```
python -m http.server 8000
```

and visit `http://localhost:8000/`.

## Build

There is no build. It is a single `index.html` with inline CSS and JavaScript and no dependencies. Edit the file and reload.

## How it works

The page reads each wall time as if it were UTC, asks the browser's `Intl.DateTimeFormat` for the zone's offset one day before and one day after, and keeps each candidate instant whose own offset maps back to the same wall time. Two survivors make a fold, none make a gap. The transition instant is found by bisection on whole seconds. Every offset keeps its seconds, so an LMT offset such as New York's -04:56:02 in 1850 gives an exact UTC value; the RFC 9557 string is omitted for those rows because the RFC 3339 offset syntax is hours and minutes only.

The results depend on the tz database release your browser carries. The page cannot tell you which release that is, so it lists the transitions it actually used. For rows before 1970 it quotes the tz database's own warning: "In short, many, perhaps most, of the tz database’s pre-1970 and future timestamps are either wrong or misleading."

## Sources

- Temporal `disambiguation` option, including "Acts like 'earlier' for backward transitions and 'later' for forward transitions.": https://tc39.es/proposal-temporal/docs/zoneddatetime.html
- PEP 495, "Mind the Gap": "the rules in effect before the transition should be used if fold=0. Otherwise, the rules in effect after the transition should be used.": https://peps.python.org/pep-0495/
- PostgreSQL Appendix B.2, "Handling of Invalid or Ambiguous Timestamps": "an invalid timestamp that appears to fall within a jump-forward daylight savings transition is assigned the UTC offset that prevailed in the time zone just before the transition, while an ambiguous timestamp that could fall on either side of a jump-back transition is assigned the UTC offset that prevailed just after the transition.": https://www.postgresql.org/docs/current/datetime-invalid-input.html
- pandas `tz_localize`, defaults `ambiguous='raise'` and `nonexistent='raise'`: https://pandas.pydata.org/docs/reference/api/pandas.Series.dt.tz_localize.html
- pandas 3.0.5 `tzconversion.pyx`: https://raw.githubusercontent.com/pandas-dev/pandas/v3.0.5/pandas/_libs/tslibs/tzconversion.pyx
- RFC 9557, the `[time-zone]` suffix (ABNF in section 4.1; example `1996-12-19T16:39:57-08:00[America/Los_Angeles]` in section 4.2), and section 2.2, which updates RFC 3339 so that `Z` means the UTC time is known but the local offset is unknown: https://www.rfc-editor.org/rfc/rfc9557
- tz database theory, pre-1970 warning: https://data.iana.org/time-zones/theory.html
- tz database rules for Eire and Lord Howe: https://data.iana.org/time-zones/tzdb/europe and https://data.iana.org/time-zones/tzdb/australasia

The New York example above, the gap and fold fixtures and the pandas cases in the self test were re-run before publishing in pandas 3.0.6, Python 3.14.7 `zoneinfo` with the `tzdata` package, PostgreSQL 18.3 (PGlite 0.5.8) and Temporal in a current Chromium-based browser.

## Third-party notices

- **pandas**, BSD 3-Clause License, https://github.com/pandas-dev/pandas. Copyright (c) 2008-2011, AQR Capital Management, LLC, Lambda Foundry, Inc. and PyData Development Team. Copyright (c) 2011-2026, Open source contributors. What was taken: the `ambiguous='infer'` grouping and switch rule (`_get_dst_hours`) and its error messages, reimplemented in JavaScript from `pandas/_libs/tslibs/tzconversion.pyx` at v3.0.5. The predictor was re-checked against pandas 3.0.6, whose `tzconversion.pyx` differs from v3.0.5 only in how the `nonexistent` argument is validated.
- PostgreSQL, Python and Temporal behaviour was measured empirically and is reimplemented from their published rules. No code from them is reproduced here.

This is an independent project. It is not affiliated with or endorsed by the pandas project, the PostgreSQL Global Development Group, the Python Software Foundation, TC39 or IANA.

## Related

- [timezone-converter](https://github.com/0xelitesystem/timezone-converter): one moment across many zones, with a meeting planner.
- [unix-timestamp-converter](https://github.com/0xelitesystem/unix-timestamp-converter): convert a single epoch value to and from a readable date.
- [timezone-and-utc-basics-reference](https://github.com/0xelitesystem/timezone-and-utc-basics-reference): plain-language reference on time zones and UTC.

## License

MIT. See [LICENSE](LICENSE).
