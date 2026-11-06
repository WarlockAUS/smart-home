# Cat Flap - Curfew with Reconcile Recovery (v2.1)

Enforces a curfew lock schedule for a Sure Petcare cat flap using Home
Assistant's native `lock` entities. Unlike a simple two-trigger
schedule, this blueprint continuously reconciles the flap's actual
lock state against what it should be for the time of day - so a
missed trigger, a dropped connection, a battery change, or an HA
restart never leaves the flap stuck in the wrong state until the next
scheduled event.

## Background

This blueprint was built after discovering that a simpler lock/unlock
schedule has a real weakness: it only acts at two fixed moments in
the day. If either of those moments is missed for any reason - the
flap was offline, Home Assistant was restarting, a service call
failed silently - there's no corrective mechanism until the next
scheduled trigger, up to 24 hours later.

This version solves that by treating the schedule as something to be
continuously *enforced* rather than *fired once*: a periodic check
compares the flap's real, current lock state (read from its `lock`
entities) against what it should be right now, and corrects any
mismatch immediately, regardless of why it occurred.

## Requirements

This blueprint is written for Home Assistant's **native core
`surepetcare` integration**, which exposes three separate lock
entities per flap:

- `lock.<name>_locked_in`
- `lock.<name>_locked_out`
- `lock.<name>_locked_all`

Only one of the three should ever be `locked` at a time; if none are
locked, the flap is unlocked. This blueprint reads and writes all
three to keep them consistent.

**This blueprint will not work with the community `pysurepetcare` /
`sureha` integrations**, which use a single `set_lock_state` service
with a `flap_id` parameter instead of per-mode lock entities. Those
integrations were evaluated during development; `sureha` was found to
offer a useful `status.locking.mode` monitoring attribute but no
service to write curfew schedules to the device itself, and its lock
control model is less granular than the native integration's. The
native integration was kept as the better, lower-risk choice.

## How it works

Seven triggers feed into one consistent decision: given the current
time and the configured curfew window, what lock mode *should* be
active right now, and does that match what's *actually* active?

| Trigger | Purpose |
|---|---|
| Curfew start time | Scheduled change to the start-of-curfew mode |
| Curfew end time | Scheduled change to the end-of-curfew mode |
| Flap reconnects | Immediate re-sync after the flap comes back online (e.g. after a battery change) |
| Reconciliation interval | Periodic safety-net check (default every 10 minutes) - catches anything the above three miss |
| Battery below threshold | First low-battery alert |
| Daily battery reminder time | Repeats the low-battery alert for as long as it stays low |
| Pet presence sensor(s) | Optional "cat in" / "cat out" notifications |

The first four triggers all route through the same logic: compute
`desired_mode` from the time of day and curfew settings, compute
`actual_mode` by reading the three lock entities, and only issue lock
commands if they differ. This means the exact same correction logic
protects against a missed scheduled trigger, a flap that was offline
at the scheduled moment, and routine drift - there's no special-cased
"catch-up" logic, just one rule applied consistently and often.

## Import

https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://github.com/<owner>/smart-home/blob/main/blueprints/automation/cat_flap_curfew.yaml

## Inputs

| Input | Description | Default |
|---|---|---|
| `lock_in_entity` | The `lock.*_locked_in` entity | — |
| `lock_out_entity` | The `lock.*_locked_out` entity | — |
| `lock_all_entity` | The `lock.*_locked_all` entity | — |
| `connectivity_entity` | `binary_sensor` reporting whether the flap is online | — |
| `curfew_start` | Time curfew begins | — |
| `curfew_end` | Time curfew ends | — |
| `start_lock_state` | Lock mode to apply at curfew start | `locked_in` |
| `end_lock_state` | Lock mode to apply at curfew end | `unlocked` |
| `reconcile_interval` | How often to run the safety-net check | Every 10 minutes |
| `presence_debounce` | How long a pet presence sensor must hold its new state before notifying | 5 seconds |
| `battery_entity` | The flap's battery level sensor | — |
| `battery_threshold` | Low battery warning threshold (%) | 20 |
| `battery_reminder_time` | Time of day the daily low-battery reminder repeats | 09:00 |
| `pet_presence_entity` | One or more pet presence `binary_sensor`s | — |
| `notify_on_curfew_start` | Optional action run when curfew starts on schedule | *(none)* |
| `notify_on_curfew_end` | Optional action run when curfew ends on schedule | *(none)* |
| `notify_on_reconnect_recovery` | Optional action run whenever a correction is made (reconnect or reconcile) | *(none)* |
| `notify_on_low_battery` | Optional action run on low battery, and daily while it remains low | *(none)* |
| `notify_on_pet_in` | Optional action run when a tracked pet comes inside | *(none)* |
| `notify_on_pet_out` | Optional action run when a tracked pet goes outside | *(none)* |

## Using `desired_mode` and `actual_mode` in notifications

Two variables are computed before any notification action runs, and
are available inside any of the optional notify inputs:

- `{{ desired_mode }}` - what the lock mode should currently be
- `{{ actual_mode }}` - what it actually was, before correction

Example message for `notify_on_reconnect_recovery`:
