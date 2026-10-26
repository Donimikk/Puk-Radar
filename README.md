# puk-radar

A personal web app for following favourite hockey players across leagues — mainly young
Slovak players, from U18 juniors to the NHL. A user signs in, adds any number of players and
sees how each of them did in their last game, with a per-game rating, recent form and
long-term development.

Status: research done, MVP spec written ([SPEC.md](SPEC.md)), nothing built yet.

## Data approach
- **Player catalogue:** Elite Prospects tells where a player currently plays (team, league, loans, transfers).
- **Stats come from each league's own source**, not from an aggregator:
  - [`sportsdataverse-py`](https://github.com/sportsdataverse/sportsdataverse-py) for ~30 leagues
    (NHL, AHL, CHL, USHL and other HockeyTech leagues, Liiga, SHL, HockeyAllsvenskan, KHL, NCAA)
  - Czech competitions: `json.esports.cz` schedule + hokej.cz public S3 shot data, hokej.cz match pages
  - Slovak competitions: hockeyslovakia.sk match and player pages
- Sites that block data-centre IPs (hockeyslovakia.sk, hokej.cz HTML) are fetched by a small
  collector running on a home network. Each finished game is fetched once.

## Rating
A single 1–10 number per game: Game Score from the inputs a league provides, turned into a
percentile within the league and position.

Project notes and the full source research live in the private `dominik-hub` repo
(`projects/Puk-Radar.md`, `research/puk-radar-zdroje.md`).
