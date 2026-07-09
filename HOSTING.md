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

## Album art failover

`widget.py` never leaves the `album_art` field blank, even when the now-playing
source has no real cover for a track. It resolves art in this order, once per track
change:

1. **The source's own art** — Spotify's own cover, or Last.fm's, if it looks real.
2. **Last.fm's placeholder is detected and skipped.** When a track genuinely has no
   art, Last.fm's API doesn't return an empty value — it returns a generic gray
   "sheriff star" placeholder image (a well-documented Last.fm API quirk, same hash
   every time). `widget.py` recognizes that specific image and treats it the same as
   no art at all, instead of showing it to everyone.
3. **[iTunes Search API](https://performance-partners.apple.com/search-api)** — free,
   no key or auth required. Looked up by artist + track name, upsized to 600×600.
4. **A static default image** — [`docs/default_album_art.png`](docs/default_album_art.png),
   served straight from this repo's raw GitHub content (works because the repo is
   public), if nothing else turns up any art at all.

This only costs an extra HTTP request when a track's own art is actually missing or
is Last.fm's placeholder — tracks with real art skip straight past it with no added
latency or API calls. If you'd rather use your own fallback image, replace
`docs/default_album_art.png` with your own file (same name) or change
`DEFAULT_ALBUM_ART_URL` near the top of `widget.py`.

---

## Tuning for faster, more accurate sync

`widget.py`'s built-in defaults (no `config.json` needed) are already tuned tight:
`poll_interval_seconds=2`, `tick_interval_seconds=0.2`, `min_patch_interval_seconds=0.4`.
These control local decision speed only and are separate from Discord's own
rate-limit bucket (`rate_limit_reserve`, covered below) — tightening them can only
reduce latency, never cause a 429, since every actual send is still gated by the
live `X-RateLimit-Remaining` check regardless of how often `main()` asks to send.

If you're on the **Last.fm** now-playing source, `poll_interval_seconds` matters more
than usual: Last.fm doesn't report a playback position the way Spotify does, so
`LastfmClient` starts its own timer the moment a poll first notices a new track —
whatever the poll cadence is becomes a fixed offset baked into that timer for the
rest of the song. The tightened default (2s) keeps that offset small while staying
far under Last.fm's ToS ceiling (~5 req/sec; this is ~0.5 req/sec).

Override any of these via GitHub repo Variables if you want to go even tighter (or
looser, e.g. to further cut request volume): `POLL_INTERVAL_SECONDS`,
`TICK_INTERVAL_SECONDS`, `MIN_PATCH_INTERVAL_SECONDS`.

## Troubleshooting

### Lyrics fall behind the song / stop updating for a while

Check `widget.log` (or the Actions run log) for a line like:

```
[ratelimit] limit=3 remaining=1 reset_after=39.0s -> next send in 39.0s
```

If `limit` is small (3 is common for widget PATCH buckets), you'll periodically hit
a stretch where Discord's own rate limit forces a wait before the next line can go
out — that's a hard ceiling on this Discord API, not a bug, and not a lyrics-lookup
failure (LRCLIB is almost certainly working fine underneath; the loop already
recomputes the true current line every tick, so whatever goes out the moment the
wait clears is always up to date, never a stale queued line).

`rate_limit_reserve` defaults to **0** — this genuinely matters and was found by
reading real logs closely: with a bucket of `limit=3`, the code used to floor this
value at a minimum of 1 no matter what you set, meaning only 2 of the 3 real tokens
were ever spent before gliding into the wait (confirmed from logs where `remaining`
alternated 2 then 1 every cycle, never reaching 0). Since Discord's own response
headers tell `widget.py` the real remaining count on every single send, there's no
need to hold a token back "just in case" on top of that — the bucket-empty case
(`remaining <= 0`) is already handled correctly by waiting the real reset window.
With `reserve=0`, all 3 tokens fire immediately before the wait kicks in: one extra
full-speed lyric update per cycle, for free, with no added 429 risk.

**Raising `rate_limit_reserve` above 0 will make lyrics fall behind faster, not
slower** — every unit above 0 makes the pacer start gliding on an earlier token.
Only raise it if you see actual `429` errors in `widget.log` (not just the normal
logged wait) — that would mean something else besides this script is also spending
the same bucket.

There is no reserve value that avoids the wait once the bucket is genuinely
exhausted — that part is a hard Discord-side ceiling, not a tunable. If your
widget's bucket is unusually small even after this fix, the only remaining lever is
reducing how often lines change (e.g. a song/section with denser lyrics will hit
the ceiling more often) — but for most widgets the widget now uses 100% of the
real bucket before ever having to wait.

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
