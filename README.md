# warera-bunker-activity-monitor

A Discord notification bot for War Era. It polls the world map every 2 hours, detects bunker and region events, and posts an alert to a Discord channel whenever something changes in a region at most **2 adjacency hops** from Ireland.

The watch set is built fresh each run: it starts from Ireland's home regions (keyed on the immutable region code prefix, so a conquered Irish region stays watched) and walks the game's `neighbors` graph outward. Because that graph is sea-aware, 2 hops from island Ireland reaches the UK, Iceland and Portugal as well as transatlantic coasts (Canada, US, Greenland), currently about 25 regions in total.

## What it tracks

Each run compares the current world state against the last saved snapshot and emits one or more events per region:

| Event | Meaning |
|---|---|
| `came_online` | Bunker started running |
| `went_offline` | Bunker stopped running |
| `level_changed` | Running level changed |
| `built` | A bunker entry appeared |
| `destroyed` | A bunker entry disappeared |
| `ownership_changed` | The region's controlling country changed |
| `construction_started` | Bunker construction began |
| `battle_started` | A battle began on the region |
| `battle_ended` | The active battle finished |
| `bunker_activating` | Bunker is pending and will activate at a scheduled time |
| `resistance_full` | An occupied region's resistance bar hit max, so a liberation battle can be started |

The game does not expose why a bunker changed state (oil exhaustion, manual disable, battle damage), so the alert states the change and leaves humans to investigate.

## What it watches

Two settings in `alert.py` control the watch set:

- `HOME_COUNTRY_CODE` (default `"ie"`) — the home country. Its regions seed the set, matched on the region code prefix (for example `ie-leinster` resolves to `ie`), which is fixed for the life of the region.
- `HOPS` (default `2`) — how many adjacency steps to expand outward over each region's `neighbors`.

Set `HOPS = 0` to watch only the home regions, `1` for home plus direct neighbors, and so on. The set is recomputed from live neighbor data every run, so it tracks the map as borders shift.

## Run modes

```bash
python alert.py              # normal run: detect transitions, post alerts
python alert.py --heartbeat  # daily status summary, no API or state changes
```

The heartbeat reports how many runs and alerts happened in the last 24 hours and warns the channel if the last successful run is older than 4 hours, which usually means the cron has stalled.

## Setup

1. Create a Discord webhook in the target channel and copy its URL.
2. Set the webhook URL as an environment variable:

```bash
export DISCORD_BUNKER_WEBHOOK_URL="https://discord.com/api/webhooks/..."
```

3. Run it. The first run snapshots the world and sends no alerts, it only seeds `state.json`. Every run after that compares against the previous snapshot.

In production the bot runs on a GitHub Actions cron every 2 hours. The workflow commits the updated `state.json` and `runs.json` back to the repo so the next run has the previous state to compare against.

## State files

Both files are committed back by the workflow and should be kept in the repo.

`state.json` holds a per-region snapshot from the last run. It is the baseline the next run compares against. Deleting it forces a fresh seed on the next run (no alerts that run).

`runs.json` is a rolling log of the last 100 runs, recording timestamp, success, event counts, and region count. The heartbeat reads it to build its summary.

## How ownership is resolved

Three fields on each region matter:

- `countryCode` is the original owner's code and never changes.
- `initialCountry` is the original owner's id and matches `countryCode`.
- `country` is the current controller's id and changes on conquest.

The current controller's code is resolved by mapping `country` against an id-to-code table built from every region's `initialCountry` to `countryCode` pairing. Alerts say "Occupied by" when the current holder is not the original owner, and "Controlled by" otherwise.

## How activation is detected

The bulk region object's `bunker.status` can be stale, and it never carries the activation timestamp. So for any monitored region that has a bunker, the bot makes one extra call to `upgrade.getUpgradeByTypeAndEntity` to read the real status and `willBeActiveAt`. A `bunker_activating` alert fires once when the bunker first becomes pending or when its activation time changes, so the same pending state across multiple polls does not re-alert.

## How resistance is handled

Resistance only climbs while a region is occupied (owner-controlled regions decay), so `resistance_full` fires only for occupied regions. Because `resistanceMax` creeps up with development, a region pinned at the cap can briefly read just under it. A hysteresis flag suppresses repeat alerts until resistance falls back below 90% of max, giving one alert per cycle.

## Reliability

The Cloudflare proxy occasionally times out or cold-starts slowly. The bot retries transient failures (HTTP 5xx, timeouts, connection errors) up to 3 times with a backoff. If a run still fails to fetch, it posts an ops message, logs the failure to `runs.json`, and exits cleanly so the cron does not spam errors. If alert delivery fails after retries, state is not saved, so the next run re-detects and retries the same alerts.

## Data source

All map data comes through the public proxy at `https://warera-proxy.toie.workers.dev/trpc`, using `region.getRegionsObject` for the bulk snapshot and `upgrade.getUpgradeByTypeAndEntity` for per-bunker activation detail.
