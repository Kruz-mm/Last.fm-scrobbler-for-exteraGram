# Last.fm Scrobbler for exteraGram

A plugin that automatically scrobbles the music you listen to in [exteraGram](https://exteragram.app/) to [Last.fm](https://www.last.fm/).

Enter your API keys, log in to your account once — and every track played in the built-in Telegram player is scrobbled from that moment on.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://github.com/Kruz-mm/Last.fm-scrobbler-for-exteraGram/blob/main/LICENSE)
[![Version](https://img.shields.io/badge/version-1.0.0-brightgreen.svg)](https://github.com/Kruz-mm/Last.fm-scrobbler-for-exteraGram)
[![exteraGram](https://img.shields.io/badge/exteraGram-%3E%3D12.5.1-2AABEE.svg)](https://exteragram.app/)
[![Python](https://img.shields.io/badge/Python-3.11-3776AB.svg)](https://plugins.exteragram.app/docs)

---

## Overview

exteraGram is a modified Telegram client for Android with a built-in Python plugin engine. This plugin hooks into the client's media controller and keeps Last.fm in sync with whatever you are playing: it sends **Now Playing** updates, counts your real listening time, and submits scrobbles according to Last.fm's own rules.

Everything is handled from a native settings page inside the app: toggles, API credentials, login, and logout. No external servers, no extra apps — the plugin talks to Last.fm directly.

## Features

- **Automatic scrobbling** — tracks playing in Telegram are scrobbled without any manual action.
- **Now Playing updates** — optionally broadcast the current track to Last.fm in real time.
- **Correct scrobble rules** — respects Last.fm's requirements: minimum length, listening threshold, and timestamp of when playback actually started.
- **Honest playtime counting** — pauses do not count towards the scrobble threshold.
- **Loop handling** — replaying the same track (for example, on repeat) counts as a new play.
- **Offline queue** — scrobbles are persisted to plugin storage and resubmitted later if the network is unavailable.
- **Batched submission** — up to 50 scrobbles per API request.
- **Smart retries** — temporary errors are retried after 30 seconds; rate limits and auth errors back off for 5 minutes.
- **Session safety** — an invalid session key is detected (Last.fm error 9) and cleared, so you can simply log in again.
- **One-time login** — the password is erased immediately after the session key is issued; only the key is stored.
- **In-app settings** — dedicated exteraGram settings page with toggles, inputs, and login/logout controls.

## Requirements

- **exteraGram 12.5.1 or newer** with the plugin engine enabled (SDK baseline 1.4.4.3).
- A **Last.fm account**.
- Your **own Last.fm API credentials** — an API key and a shared secret, created for free at [last.fm/api/account/create](https://www.last.fm/api/account/create).

## Installation

1. Download [`lastfm_scrobbler.plugin`](https://github.com/Kruz-mm/Last.fm-scrobbler-for-exteraGram/raw/main/lastfm_scrobbler.plugin).
2. In exteraGram, open **exteraGram Preferences → Plugins** and make sure the plugin engine is enabled.
3. Install the plugin by opening the `.plugin` file with exteraGram from your file manager, or by copying it into the app's plugins folder and reloading plugins.
4. Open the plugin's settings page and continue with the setup below.

## Setup

1. Create your API credentials at [last.fm/api/account/create](https://www.last.fm/api/account/create).
2. Open the **Last.fm Scrobbler** settings page in exteraGram.
3. Fill in the **API key** and **Shared secret** fields.
4. Enter your Last.fm **username** and **password**.
5. Tap **Войти** (Log in). The status under the button will change to `Вы вошли как <username>`.
6. Done. The password field is cleared automatically — it is not needed anymore.

To disconnect, tap **Выйти** (Log out): the stored session key is removed and scrobbling stops until you log in again.

> Note: the plugin's settings UI is currently in Russian.

## Settings Reference

| Setting | Default | Description |
| ------- | ------- | ----------- |
| Scrobbling enabled | On | Master switch for the plugin |
| Send "Now Playing" | On | Broadcast the current track to Last.fm while it plays |
| API key | — | Your Last.fm API key |
| Shared secret | — | Your Last.fm shared secret |
| Last.fm username | — | Used only for login |
| Last.fm password | — | Used only for login, erased afterwards |
| Log in / Log out | — | Creates or revokes the Last.fm session |

## How Scrobbling Works

The plugin polls the client's media controller every **2 seconds** and only reacts to music messages. Voice messages and round videos are ignored. A track is tracked with its actual playback position, which allows the plugin to distinguish a real new play from seeking or a loop.

| Rule | Value |
| ---- | ----- |
| Poll interval | 2 seconds |
| Minimum track length | 30 seconds (shorter tracks are never scrobbled) |
| Scrobble threshold | `min(half of track duration, 4 minutes)` |
| Unknown duration | Scrobbled after 2 minutes of actual playback |
| Pauses | Not counted towards the threshold |
| Repeats | Detected and scrobbled as a new play |

Scrobbles are queued together with the timestamp of when playback started (not when the threshold was crossed), so Last.fm receives the correct listening time.

## Reliability

- **Persistent queue** — pending scrobbles are stored in the plugin settings as JSON (up to 500 entries) and survive app restarts.
- **Batching** — up to 50 scrobbles are submitted in a single `track.scrobble` request.
- **Backoff** — on failure the plugin waits 30 seconds before retrying; on rate limiting or session/permission errors (codes 4, 9, 10, 26) it waits 5 minutes.
- **Session recovery** — an invalid session key (error 9) clears the stored session automatically instead of retrying forever.
- **Hardening** — polling runs in a daemon thread; every tick is wrapped in a try/except so a single failure can never break the plugin or the client.

## How It Works

```
┌─────────────────────────── exteraGram ───────────────────────────┐
│                                                                  │
│  media controller ──poll every 2s──▶ tracker thread              │
│      (playing track)                 │                           │
│                                      ├─▶ updateNowPlaying         │
│                                      │    (once per track)        │
│                                      └─▶ pending queue ──▶ scrobble
│                                           (persisted, batched)    │
└───────────────────────────────────┬──────────────────────────────┘
                                    │ HTTPS, signed with MD5
                                    ▼
                          ws.audioscrobbler.com/2.0
```

- All requests are signed according to the Last.fm API specification: parameters sorted alphabetically, concatenated with the shared secret, and hashed with MD5 into the `api_sig` parameter. Responses are requested in JSON.
- Used endpoints: `auth.getMobileSession` (login), `track.updateNowPlaying` (now playing), `track.scrobble` (scrobbles).
- HTTP timeout: 10 seconds per request.

## Project Structure

```
Last.fm-scrobbler-for-exteraGram/
├── lastfm_scrobbler.py      # full source code of the plugin
├── lastfm_scrobbler.plugin  # identical build, ready to install in exteraGram
└── LICENSE                  # MIT
```

The `.py` and `.plugin` files have the same content; `.plugin` is simply the extension exteraGram expects for installable plugins.

## Limitations

- Only tracks with both **artist and title metadata** can be scrobbled. Music files without tags are skipped.
- Voice messages and round videos are never scrobbled.
- Requires **exteraGram 12.5.1+**; older builds enter SDK safe mode and refuse to load plugin modules.
- Requires your own Last.fm API key — usage of the official scrobbling API is free for personal use.

## License

[MIT](https://github.com/Kruz-mm/Last.fm-scrobbler-for-exteraGram/blob/main/LICENSE) © Kruz ([@Kruz_mm](https://t.me/Kruz_mm))
