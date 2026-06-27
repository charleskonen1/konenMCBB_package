# Getting Started with konenMCBB

## Overview

`konenMCBB` is a college basketball data package for R. It brings
together data from three primary sources:

- **Bart Torvik** (`barttorvik.com`) — the industry-standard adjusted
  efficiency ratings, four factors, shooting splits, player ratings,
  and schedule data for D1 men's basketball
- **ESPN** (`site.api.espn.com`) — live and historical game data
  including box scores, play-by-play, win probability, betting lines,
  on/off rotations, and a local file database for offline season-wide
  analysis
- **Legacy sources** — Sports Reference game logs and rosters, NCAA.com
  PBP, and Covers.com injury reports

------------------------------------------------------------------------

## Installation

``` r
# Install from GitHub (requires devtools or pak)
devtools::install_github("charleskonen1/konenMCBB")
```

------------------------------------------------------------------------

## Part 1: Bart Torvik Ratings

The Torvik family of functions take a `year` argument — the *ending*
year of the season (so `year = 2025` means the 2024-25 season).

### Team Ratings (T-Rank)

``` r
library(konenMCBB)
library(dplyr)

ratings <- torvik_team_ratings(year = 2025)

# Top 25 by Barthag power rating
ratings |>
  slice_max(barthag, n = 25) |>
  select(rank, team, conf, adj_oe, adj_de, barthag, wab)

# Filter to a single conference
torvik_team_ratings(year = 2025, conf = "ACC")
```

### Player Ratings

``` r
players <- torvik_player_ratings(year = 2025)

# Best offensive contributors (min 10 games)
players |>
  arrange(desc(porpag)) |>
  select(player, team, conf, yr, porpag, adjoe, usage, bpm) |>
  head(30)

# Conference-level player leaderboard
torvik_player_ratings(year = 2025, conf = "SEC") |>
  arrange(desc(bpm)) |>
  select(player, team, bpm, obpm, dbpm, pts, reb, ast)
```

### Four Factors

``` r
ff <- torvik_four_factors(year = 2025)

# Best defenses by eFG% allowed
ff |>
  arrange(d_efg) |>
  select(team, conf, d_efg, d_to_pct, d_reb_pct, d_ftr) |>
  head(20)
```

### Shooting Splits

``` r
shooting <- torvik_shooting(year = 2025)

# Best rim-finishing offenses
shooting |>
  arrange(desc(o_rim_pct)) |>
  select(team, conf, o_rim_pct, o_rim_rate, o_three_rate)

# Best rim-protecting defenses
shooting |>
  arrange(d_rim_pct) |>
  select(team, conf, d_rim_pct, d_rim_rate) |>
  head(15)
```

### Conference Aggregates

``` r
conf <- torvik_conf_stats(year = 2025)

conf |>
  select(conf, n_teams, mean_barthag, mean_adj_oe, mean_adj_de,
         top_team, total_wab) |>
  arrange(desc(mean_barthag))
```

### Historical Snapshots

``` r
# What did teams look like on March 1st, 2025?
march1 <- timeMachine_ratings(date = 20250301)

march1 |>
  arrange(desc(barthag)) |>
  select(rank, team, conf, barthag, adjoe, adjde) |>
  head(20)

# Combine schedule + Time Machine ratings for a single day
sked_tm <- super_sked_with_timemachine(date = 20250301)
sked_tm |>
  select(game_date, team, opponent, team_barthag, opp_barthag, team_WAB)
```

------------------------------------------------------------------------

## Part 2: ESPN Game Data Pipeline

### Step 1: Configure the Database Path

``` r
espn_set_db_path("~/data/espn_db")
```

Add this to your `~/.Rprofile` to make it permanent:

``` r
options(konenMCBB.espn_db_path = "~/data/espn_db")
```

### Step 2: Discover Games

``` r
# All D1 teams with ESPN IDs and conference info
teams <- espn_teams()
teams |> filter(grepl("Duke", display_name))

# Today's scoreboard
board <- espn_scoreboard()
board |> filter(status == "final") |>
  select(away_team_name, away_score, home_score, home_team_name)
```

### Step 3: Collect Games

``` r
# Fetch all D1 games on a date and save to local DB
espn_process_day(date = "2025-01-15", season = "2024-25", sleep_sec = 0.3)

# Or fetch a single game without saving
game <- espn_game(game_id = "401703437")
names(game)
# "box_team_stats" "box_players" "game_info" "leaders"
# "winprobability" "pickcenter" "plays" "on_off"
```

### Step 4: Load Saved Games

``` r
game <- espn_load_game(game_id = "401703437", season = "2024-25",
                       date = "2024-11-11")

# Player box score
game$box_players |>
  select(display_name, team_id, minutes_numeric, points,
         true_shooting_pct, plus_minus) |>
  arrange(desc(plus_minus))

# Win probability chart data
game$winprobability |>
  select(seq_num, home_win_percentage, play_text) |>
  tail(10)

# Betting lines
game$pickcenter |>
  select(provider, spread, over_under, home_team_odds_moneyline)
```

### Step 5: Season-Wide Analysis

``` r
box <- espn_season_box("2024-25")

# Team efficiency leaders
box |>
  group_by(team_id) |>
  summarise(
    games   = n(),
    avg_eff = mean(eff, na.rm = TRUE),
    avg_efg = mean(effective_field_goal_pct, na.rm = TRUE),
    .groups = "drop"
  ) |>
  filter(games >= 10) |>
  arrange(desc(avg_eff))

players <- espn_season_players("2024-25")

# Top players by true shooting (min 15 games, 20+ min/game)
players |>
  group_by(player_id, display_name, team_id) |>
  summarise(
    games   = n(),
    avg_min = mean(minutes_numeric, na.rm = TRUE),
    avg_pts = mean(points, na.rm = TRUE),
    avg_ts  = mean(true_shooting_pct, na.rm = TRUE),
    .groups = "drop"
  ) |>
  filter(games >= 15, avg_min >= 20) |>
  arrange(desc(avg_ts))
```

### Step 6: ESPN Power Rankings

``` r
# Full adjusted-efficiency rankings for a season through a given date
rankings <- espn_rankings_summary(
  season = "2024-25",
  date   = "2025-03-01"
)

rankings |>
  select(ranking, team_name, games, wins, losses,
         adj_off_eff, adj_def_eff, adj_net_eff, relative_rating,
         resume_score) |>
  head(25)
```

------------------------------------------------------------------------

## Part 3: Schedule & Résumé Analysis

``` r
# Full super-schedule for 2025
sked <- get_super_sked(year = 2025)

# Current team résumé table (current season, all teams)
current_resume()
```

------------------------------------------------------------------------

## Part 4: Player Stats & Injuries

``` r
# Advanced player stats from Torvik (season totals/rates)
player_stats(year = 2025) |>
  arrange(desc(bpm)) |>
  head(20)

# NCAA play-by-play (NCAA.com IDs)
pbp <- ncaa_pbp("6300347")
pbp |> filter(is_home == FALSE) |> select(period, clock, description)

# NCAA player box score
box <- ncaa_player_box("6300347")
box |> arrange(desc(points)) |> select(first_name, last_name, team_id, points, assists)

# Current injury report
injuryList() |>
  filter(status %in% c("Out", "Doubtful")) |>
  select(player, team, status, injury)
```

------------------------------------------------------------------------

## Tips

- **Torvik year convention**: `year = 2025` = the 2024-25 season (ending
  year)
- **ESPN DB path**: set `options(konenMCBB.espn_db_path = ...)` in
  `~/.Rprofile`
- **Rate limiting**: use `sleep_sec = 0.3` in
  [`espn_process_day()`](../reference/espn_process_day.md) for bulk
  collection
- **Column names**: all functions return snake_case columns throughout
- **Time Machine dates**: `timeMachine_ratings()` requires YYYYMMDD
  integer format (e.g. `20250301`), and coverage starts 2014-11-13
