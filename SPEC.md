# puk-radar — build spec (MVP)

This is the brief for building the MVP. Background, sources and every verified endpoint are in the
private hub: `projects/Puk-Radar.md`, `research/puk-radar-zdroje.md`, `worklogs/puk-radar.md`.
Where this spec and the research disagree, the research file wins for facts about sources;
this file wins for what to build.

## 1. Goal
A personal web page that answers: **"How did my players do in their last game?"**

- A feed of followed players who **finished** a game in the last 24 hours, newest first.
- A player page with the last game and a game log for the current season.
- Every stat the league publishes, nothing invented. A field the league does not publish is `null`, never 0.
- One user (Dominik). No login, no rating in the MVP.

### Non-goals (MVP)
Rating, form, prospect index, login and other users, KHL, Liiga, Finnish juniors, IIHF, NHL,
notifications, a database, a backend API. They come in later phases (section 10); the data model must not
block them.

## 2. Sources in scope
| Code | Competitions | How | Runs from |
|---|---|---|---|
| `hockeytech` | AHL, USHL, WHL | `sportsdataverse-py` pinned to commit `c586665` (`sportsdataverse.hockey.ahl` etc.) | cloud |
| `cz` | ELH, Maxa liga, Liga juniorů, Extraliga dorostu (others by config) | schedule `json.esports.cz/hokejcz/scoreboard/onlajny/{YYYY-MM-DD}.json`; box score from hokej.cz match HTML `/zapas/{id}/`; shots `s3-eu-west-1.amazonaws.com/hokej.cz/visualization/shots/{id}.json` | schedule + S3: cloud; HTML: **home** |
| `sk` | Tipsport liga (`1197`), Extraliga juniorov (`1201`); others by config | `hockeyslovakia.sk` HTML: `/sk/stats/results-date/{cid}/{slug}?MatchDate=DD.MM.YYYY`, `/sk/stats/matches/{cid}/{slug}/match/{id}/Stats` | **home** |
| `ep` | player catalogue | Elite Prospects via a community scraper (`TopDownHockey_Scraper` or `eliteprospect_scraper` — pick after a live test) | home or cloud, on demand |

"home" = Dominik's home network. hockeyslovakia.sk (Cloudflare challenge) and hokej.cz HTML (nginx 403)
refuse data-centre IPs. **Do not use proxies, headless browsers or TLS impersonation to get around this.**

## 3. Architecture
```
            GitHub Actions cron (cloud)                Task Scheduler on Dominik's PC (home)
        hockeytech · cz schedule + S3 shots               sk · cz match HTML
                      │                                          │
                      └──────── git commit to data/ ─────────────┘
                                         │
                         build_site (Python) → site/ (static)
                                         │
                                 Cloudflare Pages
```
- **No database.** Data lives as JSON files in `data/`, shaped like the future tables (section 5),
  so moving to Supabase later is an import, not a rewrite.
- Every file has exactly one writer: each collector only adds or replaces its own files (for `cz` the cloud writes
  `{id}.shots.json`, home writes `{id}.json`) → the two collectors never touch the same file.
  Both do `git pull --rebase` before `git push`.
- Each finished game is fetched **once**. Requests to a source do not grow with the number of followed players.
- The site is static. The 24-hour window is applied in the browser against the current time (a few lines of JS),
  so the page is correct between builds.

## 4. Repo layout
```
puk_radar/                  Python package
  models.py                 dataclasses for section 5
  sources/
    base.py                 Source protocol (section 6)
    hockeytech.py  cz.py  sk.py  ep.py
  collect.py                CLI: python -m puk_radar.collect --where cloud|home [--date YYYY-MM-DD]
  players.py                CLI: add / confirm / list players (section 7)
  build_site.py             data/ → site/
  http.py                   shared HTTP client: timeouts, retry only on 429/5xx, per-host delay, User-Agent
templates/                  Jinja2 templates for the site
data/
  competitions.json
  players.json
  follows.json
  games/<source>/<season>/<game_id>.json
  state/<source>.json       fetched game ids, last run, last error
fixtures/                   saved server responses for tests (already has phase 0 files)
tests/                      parser tests on fixtures — no network in tests
.github/workflows/          collect.yml (cron), deploy.yml (build + Pages), test.yml
scripts/home-collect.ps1    what Task Scheduler runs on the PC
SPEC.md  README.md  WORKLOG.md (gitignored)
```
Dependencies: `sportsdataverse` (git, pinned), `httpx`, `selectolax` or `beautifulsoup4`, `jinja2`, `pytest`.
Nothing else without asking. Python 3.12+ (live test ran on 3.14).

## 5. Data model (files now, tables later)
All times are UTC ISO 8601. IDs from sources are stored as strings.

**`competitions.json`** — `[{id, source, source_competition_id, name, country, level, season}]`
- `id` = `"{source}:{source_competition_id}"`, e.g. `"sk:1197"`, `"hockeytech:ahl:2027"`.
- `level` is free text for now (`"senior"`, `"U20"`, `"U18"` …); league strength comes with the rating phase.

**`players.json`** — `[{id, name, birth_year, position, ep_id, identities: [...]}]`
- `id` = our slug, e.g. `"dominik-domonkos-2009"` (name without diacritics + birth year).
- `identities`: `[{source, external_id, competition_id, team, season, kind, confirmed}]`,
  `kind` ∈ `club | loan | national`. A player can have several identities in one season.
- Only public sports data. No contact details, no photos, nothing about minors beyond what the league publishes.

**`follows.json`** — `["dominik-domonkos-2009", ...]` (edited by hand in the MVP).

**`games/<source>/<season>/<game_id>.json`** — one file per game:
```json
{
  "id": "sk:159932", "source": "sk", "source_game_id": "159932",
  "competition_id": "sk:1197",
  "start_at": "...", "end_at": "...", "end_at_estimated": true,
  "status": "final", "home": {"name": "...", "score": 4}, "away": {...},
  "players": [ {player line}, ... ],
  "fetched_at": "...", "source_url": "..."
}
```
- `end_at`: from the source if it has one; otherwise `start_at + 150 min` and `end_at_estimated: true`.
- Store **every player in the game**, not only followed ones (the rating phase needs whole leagues).

**Player line** (`players[]`):
```
external_id, name, team ("home"|"away"), jersey, position ("F"|"D"|"G"|null),
skater: g, a, a1, a2, pts, pm, pim, sog, shot_attempts, toi_s, pp_toi_s, sh_toi_s, hits, blk, fow, fol, ppp
goalie: sa, sv, ga, sv_pct, toi_s
raw: {...every column exactly as the source sent it...}
```
Missing in the source → `null`. Times in seconds. `sv_pct` as 0–1.

## 6. Source adapters
Every adapter implements:
```python
class Source(Protocol):
    code: str
    where: Literal["cloud", "home"]
    def finished_games(self, day: date) -> list[GameRef]: ...   # games that ended on `day` (source's local date)
    def fetch_game(self, ref: GameRef) -> Game: ...             # full box score → section 5
```
Parsers are pure functions `parse_*(text_or_json) -> ...` tested on fixtures; network code stays thin.

**hockeytech** — wrap `sportsdataverse.hockey.<lg>`: `<lg>_schedule` → filter by date and final status;
`<lg>_game_summary(game_id)` → player lines. Import from the league submodule (functions are not on
`sportsdataverse.hockey`). Known: USHL has no TOI (`ice_time` 0 → store `null`), USHL PBP has no coordinates.
Wrap every call; if the library changes, only this file changes.

**cz**
- `finished_games`: onlajny scoreboard for the day; keep competitions from config; final = `match_actual_time_alias == "K"`
  (verify on more days). `hokejcz_id` is the match id.
- `fetch_game` (home): `/zapas/{hokejcz_id}/` HTML. Skater columns `Č, P, HRÁČ, TOI, PP TOI, SH TOI, G, A, B, T, +/-, H, S, B, BULY, RI`
  → jersey, position, name, toi, pp_toi, sh_toi, g, a, pts, pim, pm, hits, sog, blk, faceoffs (`"86% (6/7)"` → fow 6, fol 1), RI ignored
  (RI = `+/-` + H + blocks). Goalies `Č, P, HRÁČ, ČAS, Z, G, %Z, A, T` → toi, sv, ga, sv_pct, a, pim; `sa = sv + ga`.
  Player id from `/hrac/{slug}/{id}` links.
- Shots (cloud): S3 shots file → `shot_attempts` per `player_id` (all results), keep raw shots for later xG.
  S3 `player_id` = hokej.cz player id (verified 32/32).
- Split: the cloud collector writes schedule + shots; the home collector fills the box score into the same game file
  **only when it creates it** — to keep "one writer per file", the cloud part writes `games/cz/.../{id}.shots.json`
  and the home part writes `{id}.json`; `build_site` merges them.

**sk** (home)
- `finished_games`: `results-date` page for each configured competition and the day.
- `fetch_game`: `/Stats` page. Skaters: `Meno, #, Post, G, A, B, +/-, PIM, S, PPP, FOW` (no TOI, no FOL).
  Goalies: `Meno, #, SOGA, SVS, GA, SVS_PCT, TOI, PIM`. Player id from
  `/sk/stats/players/{cid}/{slug}/player/{id}/{name}` links. Numbers use a decimal comma (`96,30 %`).
- **Missing fixture:** a played match `/Stats` page saved from the home IP. Get it before writing the parser.

**ep** — used only by `players.py add`. Live-test both community scrapers first; record in WORKLOG which one works.
Needed fields: name, birth year, position, current team + league per season, EP id.

## 7. Players: adding and matching
`python -m puk_radar.players add "Dominik Domonkoš"`:
1. EP search → candidates (name, birth year, team, league); Dominik picks one.
2. For each current team/league that has an adapter, find the player in that source:
   hockeytech → team roster (`<lg>_team_roster`); sk/cz → player links in recent games of that team.
   Match on name without diacritics + birth year (where the source has it) + team + jersey.
3. Exact single match → identity with `confirmed: true`. Anything else → `confirmed: false`, printed for Dominik to confirm
   (`players confirm <id>`). Never guess silently.
4. League without an adapter → identity with `source: "ep"` only; the site shows "league not supported yet".

Transfers: `players.py check` (manual for now) compares the EP current team with identities and prints differences.

## 8. Collectors
- `collect --where cloud` (GitHub Actions, every 3 hours): hockeytech + cz schedule and shots, for today and yesterday.
- `collect --where home` (Task Scheduler, every 3 hours while the PC is on): sk + cz HTML, today and yesterday;
  if the PC was off, catch up to 3 days back.
- Skip game ids already in `state/<source>.json`. Re-fetch only if the stored game is not `final`.
- Politeness: one request at a time per host, ≥ 3 s between requests to hockeyslovakia.sk and hokej.cz, ≥ 1 s elsewhere;
  retry only 429/5xx with backoff, honour `Retry-After`; stop the source for the run after 3 failures in a row
  and write the error into `state`. User-Agent: `puk-radar/0.1 (personal, non-commercial)`.
- A failed source never blocks the others. The run commits whatever succeeded.
- Commit message: `data: <source> <date> (+N games)`. No commit if nothing changed.

## 9. Site
- Built by `build_site.py` with Jinja2 into `site/`, deployed to Cloudflare Pages from GitHub Actions
  (`wrangler pages deploy`, API token as a GitHub secret). Build after every data commit.
- **Feed** (`/`): followed players with a final game whose `end_at` is within 24 h of now, newest first.
  Card: name, team vs opponent, score, date, and all non-null stats of the line (skater or goalie layout).
  `end_at_estimated` games sort by the estimate. Below the feed: followed players with no game in 24 h, with their last game.
- **Player page** (`/p/<id>/`): identities (team, league, season), last game, season game log table, season totals per competition.
  Unsupported league → message plus EP link.
- Mobile first (Dominik checked hockeyslovakia.sk in an iPhone-size viewport). No tracking, no ads, no external scripts.
- Show where each number comes from (source name + link to the match page) in small text.

## 10. Phases
| Phase | Done when |
|---|---|
| B0 skeleton | layout from section 4, `pytest` green in CI, fixtures load |
| B1 parsers | sk, cz, hockeytech parsers return section-5 lines for every fixture; tests assert concrete values (e.g. JURUSIK 27 SOGA, 26 SVS, 60:00) |
| B2 collectors | one real day collected by both collectors and committed; second run adds nothing |
| B3 players | Domonkoš (USHL), Mešár or Kmec (AHL), one SK and one CZ player added via `players add` and confirmed |
| B4 site | feed and player pages live on Cloudflare Pages, built automatically after a data commit |
| later | rating (Game Score → percentile per league and position → 1–10; calibrate on last season: SK `TournamentID` history, CZ, HockeyTech), Liiga, Finnish juniors (Leijonat), IIHF, NHL, Supabase + login |

## 11. Rules for whoever builds this
- Develop parsers against `fixtures/`, not the live sites. Save new fixtures with exact server bytes (`fixtures/** -text`).
- Live requests to sk/cz only from the home IP, slowly, and never from an IP where the same browser session matters.
- No secrets in the repo; Cloudflare token only in GitHub secrets.
- Keep `WORKLOG.md` (gitignored) up to date; hub file `projects/Puk-Radar.md` is the short version.
- Before saying something works, run it and say what ran.
