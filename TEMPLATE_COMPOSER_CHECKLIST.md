# Template Composer setup checklist

## Service
- **Image**: `linuxserver/webtop:debian-xfce` (official, unmodified)
- **Icon**: matches reference, `https://raw.githubusercontent.com/XenaaLangliss/debian-railway/refs/heads/main/assets/debian.png`

## Volume
- Mount path: `/config`

## Networking
- Generate a domain, target port `3000`

## Healthcheck
- Path: **`/railway_healthz`** (underscore — NOT `/railway-healthz`, the reference's hyphenated value)
- **Confirmed via a direct test against Railway's live `serviceInstanceUpdate` API**: setting `healthcheckPath: "/railway-healthz"` is rejected with `Error in healthcheckPath - Invalid input`; the identical call with `/railway_healthz` succeeds. Railway's current validation only allows letters, numbers, `/`, and `_`.
- Restart policy: `ALWAYS` (matches reference)

## Start command
Same structure as the reference template, with one change: the injected nginx location block uses `/railway_healthz` instead of `/railway-healthz`, both in the `sed` pattern that inserts the location and in the `grep` that verifies the patch landed:

```sh
/bin/sh -c 'XD=/config/.config/xfce4/xfconf/xfce-perchannel-xml; if [ ! -f $XD/xfce4-desktop.xml ]; then mkdir -p $XD; printf %s "<?xml version=\"1.0\" encoding=\"UTF-8\"?><channel name=\"xfce4-desktop\" version=\"1.0\"><property name=\"backdrop\" type=\"empty\"><property name=\"screen0\" type=\"empty\"><property name=\"monitorselkies-primary\" type=\"empty\"><property name=\"workspace0\" type=\"empty\"><property name=\"color-style\" type=\"int\" value=\"2\"/><property name=\"image-style\" type=\"int\" value=\"0\"/><property name=\"rgba1\" type=\"array\"><value type=\"double\" value=\"0.070588\"/><value type=\"double\" value=\"0.070588\"/><value type=\"double\" value=\"0.086275\"/><value type=\"double\" value=\"1.000000\"/></property><property name=\"rgba2\" type=\"array\"><value type=\"double\" value=\"0.235294\"/><value type=\"double\" value=\"0.039216\"/><value type=\"double\" value=\"0.149020\"/><value type=\"double\" value=\"1.000000\"/></property></property></property></property></property></channel>" > $XD/xfce4-desktop.xml; chown -R 1000:1000 /config/.config; fi; sed -i "s|listen \[::\]:3000 default_server;|listen [::]:3000 default_server;\n  location = /railway_healthz { auth_basic off; return 200; }|" /defaults/default.conf; grep -q railway_healthz /defaults/default.conf || { echo "FATAL: webtop nginx config changed upstream, healthcheck patch failed" >&2; exit 1; }; exec /init'
```

## Variables
| Name | Default Value | Description (paste into Composer) | Optional |
|---|---|---|---|
| `CUSTOM_USER` | _(empty)_ | Username for the desktop's browser login. | No |
| `PASSWORD` | _(empty)_ | Password for the desktop's browser login. | No |
| `PGID` | `1000` | Group ID owning `/config`. Leave at 1000 unless you wipe the volume. | No |
| `PORT` | `3000` | Port Railway routes to. Must stay 3000, the port the image binds. | No |
| `PUID` | `1000` | User ID owning `/config`. Leave at 1000 unless you wipe the volume. | No |
| `RAILWAY_RUN_UID` | `0` | Runs the main process as root so it can set up the volume, then drops privileges. | No |
| `TITLE` | `Debian Desktop` | Browser tab title shown for the desktop. | No |
| `TZ` | `Etc/UTC` | Timezone for the desktop clock, e.g. Europe/Amsterdam. | No |

## Verified live (2026-10-16)
- `GET /railway_healthz` → `200`, no auth
- `GET /` → `401 Authorization Required` (correct — basic auth is in front of the desktop stream)
- Deploy reached `SUCCESS` in well under a minute
- Deploy logs confirm Selkies (the WebRTC streaming layer) started cleanly, falling back to software H.264 encoding with no GPU present (expected on Railway, not an error): `No hardware encoder on render node 0 (nvenc): Could not load CUDA library` followed by `Selkies 2.0.0 starting... Selkies server running on http://[::1]:8082`

## Known upstream consideration (disclosed, not fixed)
`linuxserver/docker-webtop`'s own README recommends `--shm-size=` for all desktop images, due to Chromium's known sensitivity to small `/dev/shm` in some container runtimes. Railway doesn't expose a shared-memory-size configuration option. This was not observed to cause an issue in testing, and no speculative patch was applied for it — only confirmed, verified fixes are shipped in this template's configuration. Documented in the README as a known upstream/platform interaction in case a user hits Chromium instability.
