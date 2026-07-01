# Discord-Lyrically-Widget

Realtime Spotify synced-lyrics → Discord profile widget, running as a **cloud daemon
inside GitHub Actions** — no PC, no VPS, no Docker.

Forked from [Lyrically](https://github.com/KayTwoOne/Lyrically) and patched to run
unattended on GitHub-hosted runners. All original logic (Spotify polling, LRCLIB
lyric sync, Discord rate-limit handling, album-art widget fix) is unchanged —
`widget.py` just also knows how to run under `GITHUB_ACTIONS` and exits cleanly
before the runner's timeout so the workflow can restart it.

- **One-time app/widget setup:** [SETUP.md](SETUP.md)
- **Running it as a GitHub Actions daemon:** [HOSTING.md](HOSTING.md) ← start here
- **Workflow:** [`.github/workflows/update.yml`](.github/workflows/update.yml)

## Quick start

1. Follow [SETUP.md](SETUP.md) to create the Discord app + widget and a Spotify app.
2. Run `python get_spotify_token.py` **once, locally**, to get your Spotify refresh
   token.
3. Add all six credentials as **GitHub Secrets** (see [HOSTING.md](HOSTING.md)).
4. Actions tab → **Update Discord Widget** → **Run workflow**.

The workflow runs the daemon for ~5h50m at a time, then re-triggers itself
automatically; a 6-hourly `schedule:` cron is a fallback in case the self-trigger
ever misses. You can also just manually re-run it from the Actions tab any time.
