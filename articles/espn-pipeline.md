# ESPN Data Pipeline

This article covers the full ESPN data pipeline — from fetching individual
games to building season-wide team and player databases with opponent-adjusted
efficiency rankings.

## Architecture

The pipeline follows a three-stage pattern:

```
espn_scoreboard()          # discover game IDs for a date
   ↓
espn_process_day()         # fetch + save raw + parsed game files
   ↓
espn_season_box()          # aggregate across the full season
espn_season_players()      # player-level season aggregation
espn_rankings_summary()    # opponent-adjusted power ratings
espn_conference_standings() # conference-by-conference standings
```

Games are saved as `.rds` files under a local directory tree:

```
<base_path>/
  <season>/             e.g. "2024-25"
    <YYYY-MM-DD>/       e.g. "2025-01-15"
      <game_id>/        e.g. "401703437"
        raw.json
        game.rds        parsed summary list
        box_team_stats.rds
        box_players.rds
        plays.rds
        win_prob.rds
        on_off.rds
```

------------------------------------------------------------------------

## Step 1: Configure the Database Path

```r
library(konenMCBB)

espn_set_db_path("~/data/espn_db")
# or persistently:
# options(konenMCBB.espn_db_path = "~/data/espn_db")  # in ~/.Rprofile
```

------------------------------------------------------------------------

## Step 2: Discover Available Games

```r
# Teams (with ESPN IDs, conference, colors, logos)
teams <- espn_teams()
teams |> dplyr::filter(conference_name == "ACC")

# Scoreboard for a specific date
board <- espn_scoreboard(date = "2025-01-15")
board |> dplyr::filter(status == "final") |>
  dplyr::select(game_id, away_team_name, away_score, home_score, home_team_name, venue_name)
```

------------------------------------------------------------------------

## Step 3: Collect Games

### Single game (in memory, no saving)

```r
game <- espn_game(game_id = "401703437")

# Each element is a parsed tibble/data.frame
game$box_team_stats   # team-level box score with advanced metrics
game$box_players      # player box score
game$game_info        # venue, date, status
game$winprobability   # win probability by play
game$pickcenter       # betting lines (multiple providers)
game$plays            # play-by-play with coordinates
game$on_off           # on/off rotations
```

### Full day (save to local DB)

```r
# Fetch every D1 game on a date, sleep 0.3s between requests
espn_process_day(
  date       = "2025-01-15",
  season     = "2024-25",
  sleep_sec  = 0.3
)
```

### Backfill a full season

```r
# Collect all games Nov 2024 – Apr 2025
dates <- seq(as.Date("2024-11-01"), as.Date("2025-04-08"), by = "day")
for (d in as.character(dates)) {
  espn_process_day(date = d, season = "2024-25", sleep_sec = 0.3)
  Sys.sleep(1)
}
```

------------------------------------------------------------------------

## Step 4: Load Saved Games

```r
game <- espn_load_game(
  game_id = "401703437",
  season  = "2024-25",
  date    = "2024-11-11"
)

# Player leaders sorted by true shooting
game$box_players |>
  dplyr::filter(!is.na(minutes_numeric), minutes_numeric >= 10) |>
  dplyr::arrange(dplyr::desc(true_shooting_pct)) |>
  dplyr::select(display_name, team_id, points, true_shooting_pct, plus_minus)

# Win probability chart
game$winprobability |>
  dplyr::select(seq_num, home_win_percentage, play_text) |>
  tail(5)
```

------------------------------------------------------------------------

## Step 5: Season-Wide Data

### Team box scores

```r
box <- espn_season_box("2024-25")

# Available columns (varies by game/season)
glimpse(box)

# Best eFG% by team (min 20 games)
box |>
  dplyr::group_by(team_id) |>
  dplyr::summarise(
    gp      = dplyr::n(),
    avg_efg = mean(effective_field_goal_pct, na.rm = TRUE),
    avg_eff = mean(eff, na.rm = TRUE),
    .groups = "drop"
  ) |>
  dplyr::filter(gp >= 20) |>
  dplyr::arrange(dplyr::desc(avg_efg))
```

### Player logs

```r
players <- espn_season_players("2024-25", active_only = TRUE)

# Top scorers (min 15 games, 20+ min)
players |>
  dplyr::group_by(player_id, display_name, team_id) |>
  dplyr::summarise(
    gp      = dplyr::n(),
    avg_min = mean(minutes_numeric, na.rm = TRUE),
    avg_pts = mean(points, na.rm = TRUE),
    avg_ts  = mean(true_shooting_pct, na.rm = TRUE),
    avg_pm  = mean(plus_minus, na.rm = TRUE),
    .groups = "drop"
  ) |>
  dplyr::filter(gp >= 15, avg_min >= 20) |>
  dplyr::arrange(dplyr::desc(avg_ts))
```

### Team season summary

```r
summary <- espn_team_season_summary("2024-25")
summary |>
  dplyr::arrange(dplyr::desc(avg_eff)) |>
  dplyr::select(team_id, games, avg_pts = avg_points, avg_eff,
                avg_efg = avg_effective_field_goal_pct, avg_pace)
```

------------------------------------------------------------------------

## Step 6: Power Rankings

```r
# Opponent-adjusted rankings through March 1
rankings <- espn_rankings_summary(
  season         = "2024-25",
  date           = "2025-03-01",
  use_incremental = TRUE,
  save_state      = TRUE
)

rankings |>
  dplyr::select(ranking, team_name, games, wins, losses,
                adj_off_eff, adj_def_eff, adj_net_eff,
                relative_rating, resume_score, kill_shots_per_game) |>
  head(25)
```

The `relative_rating` column is interpreted as *points per 100 possessions
better than an average D1 team*, adjusted one pass for opponent strength.

------------------------------------------------------------------------

## Step 7: Conference Standings

```r
# All conferences
all_standings <- espn_conference_standings("2024-25")

# Single conference with efficiency view
acc <- espn_conference_standings("2024-25", conf = "ACC")
acc |>
  dplyr::select(team_name, conf_record, overall_record,
                ppg, opp_ppg, point_diff,
                off_eff, def_eff, net_eff)
```

------------------------------------------------------------------------

## Tips

- **Incremental updates**: `espn_rankings_summary()` caches daily state
  files so re-running is fast — only new days are reprocessed.
- **Storage**: ~1–3 KB per `.rds` element, ~10–50 KB per game. A full
  season (~5,000 games × 8 files) is roughly 200–400 MB.
- **Rate limiting**: 0.3 s between requests is respectful; do not go faster.
- **Plays data**: Available in `game$plays` only when the game was
  processed with full PBP. Shot chart coordinates require `coordinate_x`
  and `coordinate_y` to be non-null.
