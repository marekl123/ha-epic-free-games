# Home Assistant - Epic Games Free Game Notifier

A Home Assistant project that checks the Epic Games Store for **normally paid games that are temporarily free**, avoids duplicate alerts, tracks which games you own, and lets you confirm claimed games from your phone, dashboard, Apple Watch, Wear OS watch, or a manual script run.

## Start here

**Installing this project? Read [`INSTALL.md`](INSTALL.md) from top to bottom.**

You do not need to clone this repository onto your Home Assistant server. The installation guide tells you exactly which GitHub file to copy into which Home Assistant screen.

## What it does

```text
Epic Games store endpoint
        |
        v
Find normally-paid games currently at 0
        |
        +--> Already in Owned? ----------> Ignore
        |
        +--> Already in Notified? -------> Ignore
        |
        v
ONE mobile notification for all new games
        |
        v
Store promotions in Notified as pending
        |
        +--> "Claimed all" on Android/iPhone notification
        |          OR
        +--> Dashboard / Apple Watch / Wear OS / manual script
                   |
                   v
Add pending games to Owned
Mark Notified entries completed
Repair confirmed games accidentally deleted from Owned
```

## Features

- Checks Epic Games every 6 hours.
- Detects games with an original paid price that are currently discounted to zero.
- Ignores games already in your Owned list.
- Prevents repeated alerts for the same promotion.
- Sends **one notification**, even when several new games are free at the same time.
- Supports actionable notifications on **Android and iPhone/iOS**.
- Supports quick claim confirmation from a Home Assistant dashboard.
- Supports direct script execution from **Apple Watch and Wear OS**.
- Keeps pending vs confirmed claim state in a Local To-do list.
- Repairs previously confirmed games if they are accidentally deleted from Owned.
- Allows the same game to notify again if Epic gives it away in a later promotion.

## Platform support

| Platform | What works |
| --- | --- |
| Android phone | Companion App push notification with `Claimed all`; dashboard and script also work. |
| iPhone / iOS | Companion App push notification with `Claimed all`; dashboard and script also work. |
| Apple Watch | Run the claim script directly from the Home Assistant Watch app. Notification actions can also be available when the Watch app is installed. |
| Wear OS | Run the claim script directly from the Home Assistant Wear OS app; it can be added as a favorite for quick access. |
| Desktop / browser | Use the Home Assistant dashboard tile or run the script manually. |

The project does **not** depend on watch notification buttons. The direct Home Assistant script is the reliable watch path.

## Project state model

Two Local To-do lists are used.

### `todo.epic_games_owned`

Your actual claimed-game library for this project.

**Important:** keep Owned items **incomplete / needs_action**. The checker intentionally reads incomplete items as the active owned library.

### `todo.epic_games_notified`

This is both anti-spam memory and claim state:

```text
needs_action = notification sent, claim not yet confirmed
completed    = claim confirmed
```

Both states suppress another notification for that exact promotion.

The stored key includes the promotion end date:

```text
Game title|2026-10-08T15:00:00.000Z
```

That allows the same title to notify again if Epic gives it away in a future promotion.

## Repository layout

```text
ha-epic-free-games/
├── README.md
├── INSTALL.md
├── configuration.yaml.example
├── automations/
│   ├── epic_new_free_games.yaml
│   └── epic_mark_claimed.yaml
├── scripts/
│   └── epic_claim_pending_games.yaml
├── dashboard/
│   └── epic_claim_tile.yaml
└── .gitignore
```

There are deliberately only **two documentation files**:

- `README.md` - overview and project design.
- `INSTALL.md` - complete installation, phone/watch setup, testing, and troubleshooting.

## Components

| Component | Home Assistant location | Required |
| --- | --- | --- |
| `configuration.yaml.example` | Merge into HA `configuration.yaml` | Yes |
| `automations/epic_new_free_games.yaml` | New automation | Yes |
| `automations/epic_mark_claimed.yaml` | New automation | Yes |
| `scripts/epic_claim_pending_games.yaml` | New script | Yes |
| Owned Local To-do list | Created through HA UI | Yes |
| Notified Local To-do list | Created through HA UI | Yes |
| Template Binary Sensor | Created through HA UI | Recommended |
| `dashboard/epic_claim_tile.yaml` | Dashboard Tile card | Optional, recommended |
| Apple Watch / Wear OS script shortcut | Companion app/watch configuration | Optional |

Every one of these is explained step by step in [`INSTALL.md`](INSTALL.md).

## Important notes

- This project uses Epic's public store backend endpoint. It is **not** an official Epic Games or Home Assistant integration.
- Epic may change the endpoint or JSON response format in the future.
- The included REST example is configured for the Netherlands (`NL`). Change the region in the URL if needed.
- Your personal Epic library is intentionally **not** included. Populate the Owned list with your own games.
- Never publish Home Assistant secrets, tokens, `.storage`, `secrets.yaml`, backups, databases, or private configuration files.

## Official Home Assistant references

- Actionable notifications: https://companion.home-assistant.io/docs/notifications/actionable-notifications/
- Local To-do: https://www.home-assistant.io/integrations/local_todo/
- To-do entities/actions: https://www.home-assistant.io/integrations/todo/
- Template helpers: https://www.home-assistant.io/integrations/template/
- Apple Watch: https://companion.home-assistant.io/docs/apple-watch/
- Wear OS: https://companion.home-assistant.io/docs/wear-os/
