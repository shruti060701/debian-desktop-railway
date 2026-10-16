# Debian Desktop (XFCE) on Railway

Deploy a real XFCE desktop — not a web terminal, an actual GUI desktop with a file manager, terminal, and Chromium — streamed to your browser over WebRTC via [Selkies](https://github.com/selkies-project/selkies). Built on the official [`linuxserver/webtop:debian-xfce`](https://github.com/linuxserver/docker-webtop) image (GPL-3.0), unmodified.

## The one real fix: a healthcheck path Railway's own platform rejects

The official reference template configures `healthcheckPath: /railway-healthz`. Tested this directly against Railway's own live service-update API (the same call any deploy goes through) and it is rejected outright:

```
Error in healthcheckPath - Invalid input
```

Railway's current healthcheck path validation only allows letters, numbers, `/`, and `_` — **no hyphens**. `/railway-healthz` contains one. This template uses `/railway_healthz` instead (underscore), confirmed to pass the identical validation call, and updates the nginx location block the start command injects to match. Verified live: `GET /railway_healthz` returns `200` with zero authentication required, exactly as the design intends.

This single-character difference plausibly explains the reference template's 92 (not 100) health score — a healthcheck path that the platform's own validation rejects can't reliably apply to every deploy.

## What else was checked, honestly

Unlike some templates in this series, this reference is otherwise well-engineered — the image's own `docker-entrypoint`-equivalent init already handles volume permissions correctly (chowns `/config` to the unprivileged user before dropping root), and `RAILWAY_RUN_UID=0` is the documented, correct way to let Railway's LSIO-style images manage that permission drop themselves. No other configuration bug was found.

One upstream-documented consideration worth knowing, not something broken by this template specifically: [`linuxserver/docker-webtop`'s own README](https://github.com/linuxserver/docker-webtop#parameters) recommends `--shm-size=` for all desktop images, because Chromium is known to behave poorly under a small `/dev/shm` in some container runtimes. Railway's platform doesn't currently expose a way to configure shared-memory size. This wasn't observed to cause a problem in testing, but if you see Chromium tabs crashing inside the desktop, this is the documented reason why, and it's an upstream/platform interaction rather than anything this template's configuration controls.

## What you get

- The official XFCE Debian desktop image, GUI and all, unmodified
- A persistent volume at `/config` — settings, files, and desktop state survive redeploys
- A working, verified healthcheck at `/railway_healthz`
- Password-protected access (HTTP Basic Auth in front of the desktop stream)

## After deploying

Open the service's public domain. You'll be prompted for the `CUSTOM_USER` / `PASSWORD` you set, then land in a real XFCE desktop running in your browser.

## Variables

| Variable | Default | Purpose |
|---|---|---|
| `CUSTOM_USER` | _(required)_ | Browser login username |
| `PASSWORD` | _(required)_ | Browser login password |
| `PUID` / `PGID` | `1000` | User/group owning `/config` — leave at default unless you know you need to change it |
| `PORT` | `3000` | Must stay `3000`, the port the image binds |
| `RAILWAY_RUN_UID` | `0` | Runs the init process as root so it can correctly set up the volume, then drops to `PUID`/`PGID` |
| `TITLE` | `Debian Desktop` | Browser tab title |
| `TZ` | `Etc/UTC` | Timezone for the desktop clock |

## License

`linuxserver/webtop` is [GPL-3.0](https://github.com/linuxserver/docker-webtop/blob/master/LICENSE). This template deploys the official image, unmodified.
