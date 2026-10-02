# Installation guide

This is the complete installation guide. Follow it from top to bottom.

You do **not** need to clone this repository onto the Home Assistant server. Open each file on GitHub, copy its contents, and paste it into the matching Home Assistant editor described below.

## Installation checklist

```text
[ ] 1. Add Epic REST command
[ ] 2. Restart Home Assistant and test the REST command
[ ] 3. Create Owned Local To-do list
[ ] 4. Create Notified Local To-do list
[ ] 5. Add Epic checker automation
[ ] 6. Add claim automation
[ ] 7. Add claim script
[ ] 8. Create pending-games binary sensor
[ ] 9. Add dashboard tile (optional, recommended)
[ ] 10. Configure Android/iPhone notifications
[ ] 11. Configure Apple Watch or Wear OS (optional)
[ ] 12. Test the complete workflow
```

---

# 0. Requirements

You need:

- Home Assistant.
- Internet access from Home Assistant to the Epic Games store endpoint.
- The **Local to-do** integration.
- The Home Assistant Companion App on Android or iOS if you want phone push notifications.
- Apple Watch or Wear OS only if you want a watch shortcut; neither is required.

The supplied YAML expects these entity IDs:

```text
todo.epic_games_owned
todo.epic_games_notified
binary_sensor.epic_games_unclaimed
automation.epic_games_mark_game_as_claimed
script.epic_games_claim_pending_games
```

Home Assistant normally generates these IDs from the names used below. If your IDs differ, replace the corresponding IDs in the YAML files.

---

# 1. Add the Epic REST command

Open this repository file:

```text
configuration.yaml.example
```

Copy its `rest_command` configuration into your Home Assistant `configuration.yaml`.

The included example is:

```yaml
rest_command:
  epic_free_games:
    url: "https://store-site-backend-static.ak.epicgames.com/freeGamesPromotions?locale=en-US&country=NL&allowCountries=NL"
    method: GET
    headers:
      accept: "application/json"
    timeout: 15
```

## If you already have `rest_command:`

Do **not** create a second top-level `rest_command:` key. Add only the `epic_free_games:` block underneath your existing one.

## Region

The supplied URL uses:

```text
locale=en-US
country=NL
allowCountries=NL
```

Change the country values if you need a different Epic Games Store region.

---

# 2. Restart Home Assistant and test the REST command

When adding `rest_command` for the first time, perform a **full Home Assistant restart**. A quick reload may not register a newly-added REST command.

After Home Assistant is back:

```text
Settings -> Tools -> Actions
```

Run:

```yaml
action: rest_command.epic_free_games
```

A successful request should return a large response containing:

```text
status: 200
```

The large response is normal.

---

# 3. Create the Owned Local To-do list

If Local to-do is not installed:

```text
Settings -> Devices & services
-> Add integration
-> Local to-do
```

Create a list named:

```text
Epic Games - Owned
```

Expected entity:

```text
todo.epic_games_owned
```

This list is your owned-game library for the project.

## Important Owned-list rule

Keep games in this list **incomplete / needs_action**.

That may feel backwards for a to-do list, but it is intentional: the automation reads incomplete items as the active Owned library.

If you mark an Owned game Completed, the checker temporarily treats it as not owned.

You can now manually add your existing Epic games to this list. Your personal library is not included in this repository.

---

# 4. Create the Notified Local To-do list

Create another Local To-do list named:

```text
Epic Games - Notified
```

Expected entity:

```text
todo.epic_games_notified
```

This list has real workflow state:

```text
needs_action = notification has been sent, claim still waiting for confirmation
completed    = you confirmed the game was claimed
```

Both statuses suppress another notification for the same promotion.

Entries use a promotion-specific key similar to:

```text
System Shock 2: 25th Anniversary Remaster|2026-10-08T15:00:00.000Z
```

The end date makes the key unique to that giveaway. If Epic gives away the same title again later, it can notify you again.

---

# 5. Add the Epic checker automation

Open the repository file:

```text
automations/epic_new_free_games.yaml
```

In Home Assistant:

```text
Settings -> Automations & scenes -> Automations
-> Create automation
-> Create new automation
-> Open YAML editor
```

Paste the complete file.

## Set your phone notification action

The file contains this placeholder:

```text
notify.mobile_app_your_phone
```

Replace it with the notification action created by your own Home Assistant Companion App, for example:

```text
notify.mobile_app_pixel_9
```

or:

```text
notify.mobile_app_my_iphone
```

You can test your own notification action under:

```text
Settings -> Tools -> Actions
```

Example:

```yaml
action: notify.mobile_app_your_device
data:
  title: Home Assistant test
  message: Test notification
```

## What this automation does

Every 6 hours it:

1. calls the Epic store endpoint
2. finds games whose original price is greater than zero and current discount price is zero
3. requires an active Epic promotional offer
4. ignores titles already in Owned
5. ignores promotion keys already in Notified, regardless of whether they are pending or completed
6. sends **one phone notification** containing all newly discovered titles
7. stores every newly-notified promotion in Notified as `needs_action`

If two games are new at the same time, you receive one notification listing both games.

---

# 6. Add the claim automation

Open:

```text
automations/epic_mark_claimed.yaml
```

Create a second automation in Home Assistant and paste the complete YAML.

This automation listens for the mobile notification action:

```text
EPIC_CLAIMED_ALL
```

When triggered, it:

1. reads the Owned list
2. reads pending Notified entries
3. reads completed Notified history
4. adds pending titles to Owned if they are missing
5. changes pending Notified entries to Completed
6. restores previously-confirmed games that were accidentally deleted from Owned

Expected automation entity:

```text
automation.epic_games_mark_game_as_claimed
```

**Check the actual entity ID after saving.** You need it in the next step.

---

# 7. Add the claim script

Open:

```text
scripts/epic_claim_pending_games.yaml
```

In Home Assistant:

```text
Settings -> Automations & scenes -> Scripts
-> Create script
-> Open YAML editor
```

Paste the complete YAML.

Expected script entity:

```text
script.epic_games_claim_pending_games
```

The supplied script calls:

```text
automation.epic_games_mark_game_as_claimed
```

If Home Assistant gave your claim automation a different entity ID, edit the script target accordingly.

## Why the script exists

The phone notification triggers the claim automation directly.

The script is a reusable button entry point for:

- dashboard
- Apple Watch
- Wear OS
- manual recovery after dismissing a notification

The claim logic stays in one automation; the script simply triggers that automation's actions.

---

# 8. Create the pending-games binary sensor

This helper is recommended for the red/grey dashboard indicator.

Go to:

```text
Settings -> Devices & services -> Helpers
-> Create helper
-> Template
-> Binary sensor
```

Name:

```text
Epic games unclaimed
```

State template:

```jinja
{{ states('todo.epic_games_notified') | int(0) > 0 }}
```

Leave **Device class** empty.

Expected entity:

```text
binary_sensor.epic_games_unclaimed
```

Why it works: a Home Assistant To-do entity's state is the number of incomplete items. Therefore this binary sensor is `on` whenever one or more Notified items are still pending confirmation.

---

# 9. Add the dashboard tile

Open:

```text
dashboard/epic_claim_tile.yaml
```

Edit the Home Assistant dashboard where you want the control and add a **Tile** card. Open the card's code editor and paste the YAML.

The included tile:

- shows a gamepad icon and text on one line
- is grey when there are no pending claims
- becomes red when pending games exist
- runs `script.epic_games_claim_pending_games` when tapped
- uses `2 columns x 1 row` in a Sections dashboard

If two columns is too narrow for the text, increase:

```yaml
grid_options:
  columns: 3
  rows: 1
```

---

# 10. Phone notifications - Android and iPhone/iOS

The core automation is the same on both platforms.

The notification contains one action:

```text
Claimed all
```

Pressing it fires Home Assistant's `mobile_app_notification_action` event and triggers the claim automation.

## Android

Home Assistant actionable notifications are supported on Android. Android supports up to three notification actions; this project uses only one.

Depending on Android version and manufacturer UI, you may need to expand the notification to reveal `Claimed all`.

If your `notify.mobile_app_...` action is missing:

1. make sure the Home Assistant Companion App is connected to the correct server
2. make sure notification permission is enabled
3. force-stop/reopen the Companion App if necessary
4. restart Home Assistant so the mobile notify action can register

## iPhone / iOS

The same `Claimed all` action works on iOS.

On the lock screen or Notification Center, you may need to **press and hold** the Home Assistant notification to reveal the action.

If you accidentally tap or dismiss the notification, do not delete anything from Notified. Use the dashboard tile, Apple Watch, or manually run the claim script instead.

## Multiple new games

The checker deliberately sends **one** notification containing all new games.

`Claimed all` confirms all currently pending Notified items together.

---

# 11. Apple Watch - optional

The Home Assistant Apple Watch app can expose Home Assistant entities including scripts.

On the iPhone Home Assistant app:

```text
Settings
-> Companion app
-> Apple Watch
-> Configuration
-> Add item
-> Entity
```

Select:

```text
Epic Games - Claim pending games
```

Save, then reopen Home Assistant on the Watch.

If a newly-created script does not appear in the entity picker, fully quit the Home Assistant app on the iPhone, reopen it, and search again. This refreshes the available entity list in many cases.

Home Assistant notification actions on watchOS require the Home Assistant Watch app to be installed. The project does not rely on that, however: running the claim script directly from the Watch is the recommended shortcut.

---

# 12. Wear OS - optional

The Home Assistant Wear OS app supports executing `script` entities directly.

Initial setup requires a paired Android phone with the Home Assistant app so you can sign in to your Home Assistant server.

On the Watch, open Home Assistant and find:

```text
Epic Games - Claim pending games
```

Run it to confirm all pending games.

For faster access, add the script as a **favorite** in the Wear OS Home Assistant app. Favorites appear near the top and can be executed quickly.

The project intentionally does not depend on notification action buttons appearing on Wear OS. The direct script is the recommended watch workflow.

---

# 13. Test the complete workflow

## A. Basic no-notification test

If all currently-free Epic games are already present in Owned, manually run:

```text
Epic Games - New free paid games
```

Expected result:

```text
No notification
```

That confirms the Owned filter is suppressing known games.

## B. One-game test

1. Pick one game that is currently free and already exists in Owned.
2. Remove it temporarily from Owned.
3. If a matching current-promotion entry already exists in Notified, remove that test entry too.
4. Manually run `Epic Games - New free paid games`.
5. Verify one phone notification arrives.
6. Reveal and press `Claimed all`, or run `Epic Games - Claim pending games`.
7. Verify the game has been added back to Owned.
8. Verify the corresponding Notified item is now Completed.

## C. Two-game test

1. Temporarily remove two currently-free games from Owned.
2. Remove their matching current-promotion entries from Notified if necessary.
3. Run the checker manually.
4. Verify you receive **one notification listing both games**.
5. Confirm the claim using `Claimed all` or the claim script.
6. Verify both games are in Owned.
7. Verify both Notified entries are Completed.

## D. Anti-spam test

Run the checker again.

Expected:

```text
No new notification for the same promotion
```

The current promotion is already represented in Notified and/or the game is already in Owned.

## E. Dismissed-notification recovery test

1. Make a test game pending.
2. Dismiss the notification without pressing `Claimed all`.
3. Run `Epic Games - Claim pending games` from the dashboard, watch, or Scripts page.
4. Verify the pending game is added to Owned and its Notified item becomes Completed.

No title needs to be typed manually because the system already stored the pending game in Notified.

## F. Repair test

1. Choose a game that has a **Completed** entry in Notified.
2. Delete that game from Owned.
3. Run `Epic Games - Claim pending games`.
4. Verify the game is restored to Owned from confirmed claim history.

---

# Troubleshooting

## `rest_command.epic_free_games` is not found

Do a **full Home Assistant restart** after adding the REST command to `configuration.yaml`.

Then retest under:

```text
Settings -> Tools -> Actions
```

## Epic API returns a huge response

Expected. Look for:

```text
status: 200
```

The automation extracts only the catalog, price, and promotion data it needs.

## No phone notification arrives

Check that `automations/epic_new_free_games.yaml` uses your real notification action:

```text
notify.mobile_app_<your_device>
```

Test that notification action manually under Settings -> Tools -> Actions.

Also check Companion App notification permission.

If the mobile notify action itself is missing, reopen/force-stop the Companion App and restart Home Assistant.

## `Claimed all` is not visible on iPhone

Press and hold the notification to expand its actions.

## `Claimed all` is not visible on Android

Expand the notification. Presentation varies by Android version and manufacturer.

## I dismissed the notification accidentally

Do **not** remove the pending Notified entry.

Run:

```text
Epic Games - Claim pending games
```

The pending Notified records already contain enough information for the system to complete the workflow.

## The game is still in Notified after claiming

Correct. It should remain there but change from:

```text
needs_action
```

to:

```text
completed
```

Completed entries are claim history and anti-spam memory.

## The dashboard tile is not red

Check:

```text
binary_sensor.epic_games_unclaimed
```

It should be `on` when `todo.epic_games_notified` has at least one incomplete item.

Check the template:

```jinja
{{ states('todo.epic_games_notified') | int(0) > 0 }}
```

## Apple Watch cannot find the script

Fully quit the Home Assistant app on the iPhone, reopen it, then return to Apple Watch Configuration and search again.

## Wear OS cannot find the script

Confirm the Wear OS app is logged into the same Home Assistant server. The Home Assistant Wear OS app supports the `script` domain.

## I get duplicate Owned entries

Remove duplicate entries manually and leave one active copy of each game.

Do not normally mark Owned games Completed; keep them incomplete / needs_action.

## A game is free again months later

Supported. The Notified key includes the promotion end date, so a later giveaway creates a different key and can notify again.

---

# Official references

- Actionable notifications: https://companion.home-assistant.io/docs/notifications/actionable-notifications/
- Companion notification troubleshooting: https://companion.home-assistant.io/docs/troubleshooting/faqs/
- Local To-do: https://www.home-assistant.io/integrations/local_todo/
- To-do entities/actions: https://www.home-assistant.io/integrations/todo/
- Template helpers: https://www.home-assistant.io/integrations/template/
- Apple Watch: https://companion.home-assistant.io/docs/apple-watch/
- Wear OS: https://companion.home-assistant.io/docs/wear-os/
