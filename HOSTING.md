# Hosting Lyrically on GitHub Actions (cloud, no PC/VPS required)

This fork runs `widget.py` **entirely inside GitHub Actions** as a long-lived cloud
daemon. There is no local process, no VPS, no Docker, no separate host — GitHub's own
runners are the "server."

`widget.py` reads your playback from the **Spotify Web API** (cloud, tied to your
account, not your local app), so it sees what you're playing on any device and needs
no machine of yours to stay on.

---

## How it works

```
GitHub Actions (workflow_dispatch / cron fallback)
        │
        ▼
  ubuntu-latest runner starts
        │
        ▼
  pip install -r requirements.txt
        │
        ▼
  python widget.py            (env vars come from repo Secrets)
        │
        ├─ poll Spotify every few seconds
        ├─ fetch synced lyrics from LRCLIB
        ├─ track playback position between polls
        ├─ PATCH the Discord widget on every lyric-line change
        │
        ▼
  runs for ~5h50m (MAX_RUNTIME_SECONDS = 21000), then widget.py
  returns cleanly — well inside the job's 360-minute limit
        │
        ▼
  workflow re-dispatches itself (`Trigger next run` step) so a fresh
  runner picks up immediately; the `schedule:` trigger is a fallback
  in case the self-trigger ever fails to fire
```

Nothing here needs a persistent server: every run is a fresh runner, all state
(current track/lyrics) lives in `widget.py`'s process memory for that run, and the only
thing that needs to persist across runs is your Spotify refresh token — which lives in
a GitHub Secret, not on disk.

---

## One-time setup

### 1. Finish the Discord + Spotify app setup

Follow [SETUP.md](SETUP.md) (Parts 1–7, or the Express `lyrically-setup.js` path) to
create the Discord application, publish + authorize the widget, and create a Spotify
app. You'll end up with:

- `DISCORD_APPLICATION_ID`
- `DISCORD_USER_ID`
- `DISCORD_BOT_TOKEN`
- `SPOTIFY_CLIENT_ID`
- `SPOTIFY_CLIENT_SECRET`

### 2. Get your Spotify refresh token (local, one-time only)

The OAuth flow needs a browser + a loopback redirect, which a runner can't do, so this
one step runs on your own machine, once:

```bash
pip install -r requirements.txt
python get_spotify_token.py
```

Copy the `SPOTIFY_REFRESH_TOKEN` it prints at the end. You never need to run this
script in GitHub Actions or run `widget.py` locally.

### 3. Add the six values as GitHub Secrets

In this repo: **Settings → Secrets and variables → Actions → New repository secret**.
Add each of:

| Secret | Value |
|---|---|
| `DISCORD_APPLICATION_ID` | your Discord application ID |
| `DISCORD_USER_ID` | your Discord user ID |
| `DISCORD_BOT_TOKEN` | your Discord bot token |
| `SPOTIFY_CLIENT_ID` | your Spotify client ID |
| `SPOTIFY_CLIENT_SECRET` | your Spotify client secret |
| `SPOTIFY_REFRESH_TOKEN` | the token printed in step 2 |

Optional: `DISCORD_IMAGE_WEBHOOK_URL` if you use the album-art widget-fix feature
(see SETUP.md's Tuning section) — a Discord webhook URL used to re-host reshaped
cover art.

**No `config.json` is used or needed in this mode.** `widget.py` already prefers
environment variables over the config file (see `_ENV_MAP` near the top), so the repo
secrets are picked up automatically.

### 4. Start the daemon

Go to **Actions → Update Discord Widget → Run workflow** (`workflow_dispatch`). Watch
the run's log — you should see:

```
Started under GitHub Actions. Runtime budget: 21000s (will exit cleanly before then so the workflow can restart it).
Now playing: <song> — <artist>
Loaded N synced lyric lines.
♪ <current line>
```

Play something on Spotify (any device) — within a few seconds your Discord profile
widget updates.

---

## Keeping it running

- The workflow's last step re-dispatches itself as soon as `widget.py` returns
  (i.e. as soon as it hits its own `MAX_RUNTIME_SECONDS` budget), so a new runner
  should already be starting before the old one fully exits.
- The `schedule: cron: "0 */6 * * *"` trigger is a safety fallback in case a
  self-trigger ever fails to fire (e.g. a transient GitHub API error) — worst case
  you get a gap of a few hours before it recovers on its own.
- `concurrency: group: lyrically-widget-daemon` makes sure only one instance is ever
  PATCHing the widget at a time, so a schedule firing mid-run can't collide with the
  self-triggered run.
- If a run ever fails outright (bad credentials, unhandled exception), the job shows
  red in the Actions tab and stops self-triggering. Fix the secret/issue and click
  **Run workflow** again manually — you don't need perfect 24/7 uptime; a manual
  restart is fine.

## Rotating credentials

- **Discord bot token:** Developer Portal → your app → **Bot** → **Reset Token**,
  then update the `DISCORD_BOT_TOKEN` secret.
- **Spotify:** Developer Dashboard → your app → **Settings** → regenerate the
  **Client Secret**, then re-run `python get_spotify_token.py` locally and update the
  `SPOTIFY_REFRESH_TOKEN` secret.

Both tokens are deliberately low-privilege (widget-only write access on Discord,
read-only playback scopes on Spotify), so rotating either instantly neutralizes any
leaked copy.
