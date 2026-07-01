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

## Troubleshooting

### Lyrics look "stuck" — real text shows up but only after skipping a song or two

Check `widget.log` (or the Actions run log) for a line like:

```
[ratelimit] limit=3 remaining=1 reset_after=39.0s -> next send in 39.0s
```

If `limit` is small (3 is common for widget PATCH buckets), the pacing logic can
burst 2 lines out immediately and then freeze for the *entire* remaining window
before the next line can go out — that reads exactly like "stuck" lyrics that
mysteriously catch up later. This is a Discord-side rate-limit, not a lyrics-lookup
failure (LRCLIB is almost certainly working fine underneath).

The default `rate_limit_reserve` (2) already gives some headroom against this, but
if you still see it, raise it further via the `RATE_LIMIT_RESERVE` repo variable
(Settings → Secrets and variables → Actions → Variables tab) — e.g. `3` — so the
pacing starts gliding one token earlier and spends the bucket evenly across the
reset window instead of bursting then freezing.

### Every Spotify call 403s: `Forbidden for url: .../currently-playing`

This is **not** a bug or a misconfigured secret — as of Spotify's Feb 2026 Development
Mode policy, every Spotify app starts in Development Mode, and Development Mode now
requires **the app owner to have an active Spotify Premium subscription**. On a Free
account, auth still succeeds (you get an access token) but every playback-data call
403s. There is no code workaround for this against the official Spotify Web API —
Extended Quota Mode (which drops the requirement) is now restricted to companies with
250k+ MAU, not available to individual developers.

**If you're on Spotify Free**, switch the now-playing source to Last.fm instead —
`widget.py` supports this natively via `NOWPLAYING_SOURCE=lastfm`, and it works on any
Spotify plan since it reads your scrobbles rather than calling Spotify's API:

1. Get a free API key at the [Last.fm API account page](https://www.last.fm/api/account/create)
   (any app name/description works) — this gives you a `LASTFM_API_KEY`.
2. Make sure scrobbling is enabled from whatever you use to play music (Spotify's own
   Last.fm Connect, or a scrobbler app) so `user.getrecenttracks` reflects what's
   actually playing.
3. Add two more GitHub Secrets: `LASTFM_API_KEY` and `LASTFM_USERNAME` (your Last.fm
   username).
4. Set the repo variable `NOWPLAYING_SOURCE` to `lastfm` — **Settings → Secrets and
   variables → Actions → Variables tab** (not Secrets; it isn't sensitive) → **New
   repository variable**.
5. Re-run the workflow. You no longer need any `SPOTIFY_*` secrets in this mode.

Trade-off: Last.fm's API doesn't expose exact playback position the way Spotify's
does, so `widget.py` approximates it — it starts a local timer the moment it first
sees a track go "now playing" and treats that as position 0. If the workflow starts
mid-song, the lyric sync will carry a constant offset equal to how far into the song
it already was. Track/artist/album/art and Discord PATCH behavior are unaffected.

### Self-trigger step fails: `Resource not accessible by integration`

GitHub defaults `GITHUB_TOKEN` to read-only unless the workflow explicitly requests
more. The workflow in this repo already declares:

```yaml
permissions:
  actions: write
  contents: read
```

at the top level, which is what lets the last step's `POST .../dispatches` call
succeed. If you copy this workflow elsewhere and see this error again, that
`permissions:` block is almost always the missing piece — no PAT is required, despite
some older guides suggesting otherwise (`workflow_dispatch`/`repository_dispatch` are
explicit exceptions to GitHub's anti-recursion rule on `GITHUB_TOKEN`).

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
