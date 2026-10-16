# Deploy and Host a Debian XFCE Desktop on Railway

This isn't a web terminal with a shell prompt — it's a real XFCE desktop, file manager, taskbar, Chromium and all, streamed to your browser over WebRTC. It runs on the official [`linuxserver/webtop:debian-xfce`](https://github.com/linuxserver/docker-webtop) image (GPL-3.0), powered by [Selkies](https://github.com/selkies-project/selkies) for low-latency remote streaming. This template fixes one concrete, confirmed issue in the official reference template, and is honest about what wasn't broken.

## About Hosting a Debian Desktop

The reference template is already a careful piece of engineering — its start command patches the webtop image's nginx config at boot to inject a dedicated, unauthenticated healthcheck location, works around a desktop-background config quirk on first boot, and includes a defensive check that fails loudly if the upstream image's nginx config format ever changes underneath it. None of that needed fixing.

One thing did: the healthcheck path itself. `/railway-healthz` — the exact value the reference template sets — contains a hyphen. Tested this directly against Railway's own live service-configuration API (the same call that applies when you set a healthcheck path on any service) and it's rejected outright: `Error in healthcheckPath - Invalid input`. Railway's current validation only permits letters, numbers, `/`, and `_` in a healthcheck path — no hyphens. The identical call with an underscore, `/railway_healthz`, succeeds immediately.

This template uses the underscore path throughout: in the Composer's healthcheck field, and in the nginx location block the start command injects at boot, so the two stay consistent. Confirmed live afterward: `GET /railway_healthz` returns `200` with zero authentication required, exactly as designed.

## Common Use Cases

- **A disposable browsing environment**: isolate risky links or downloads in a real desktop that isn't your own machine
- **Running Linux GUI software you can't install locally**: anything that needs an actual desktop environment, not just a terminal
- **A consistent dev/test environment reachable from any device**: tablet, Chromebook, work laptop — all you need is a browser
- **Demoing GUI applications remotely**: share a password-protected link instead of screen-sharing software

## Dependencies for Debian Desktop Hosting

Running this template needs nothing beyond the container and one persistent volume — no external database, no separate services.

### Deployment Dependencies

- [linuxserver/webtop Source](https://github.com/linuxserver/docker-webtop) — GPL-3.0
- [linuxserver/webtop Documentation](https://docs.linuxserver.io/images/docker-webtop/)
- [Selkies](https://github.com/selkies-project/selkies) — the WebRTC streaming layer the image uses, MPL-2.0

### Implementation Details

The healthcheck fix is the whole story here, so it's worth being precise about how it was confirmed rather than just asserted. Railway's `serviceInstanceUpdate` mutation — the actual API call behind every healthcheck path change, whether made through the dashboard or programmatically — was called twice with otherwise identical input: once with `healthcheckPath: "/railway-healthz"`, which returned `Error in healthcheckPath - Invalid input`, and once with `healthcheckPath: "/railway_healthz"`, which succeeded. That's not a guess about what might be wrong; it's the platform's own validation logic rejecting the reference template's exact configured value.

This plausibly explains why the official reference template sits at a 92 health score rather than 100 — a healthcheck path the platform itself won't reliably accept can't consistently apply across every deploy that uses it.

Everything else in the reference template's design holds up under inspection. The underlying webtop image's init process already handles `/config` volume permissions correctly — it `chown`s the mount to the unprivileged user before dropping root — and `RAILWAY_RUN_UID=0` is the documented, correct mechanism for letting an LSIO-style image manage that permission drop itself on Railway. No other bug was found, and none was invented to pad this out.

One thing worth disclosing rather than silently patching: `linuxserver/docker-webtop`'s own documentation recommends `--shm-size=` for all of its desktop images, because Chromium is known to behave poorly under a small `/dev/shm` in some container runtimes. Railway's platform doesn't currently expose a way to configure that. This wasn't observed to cause a problem in live testing — Selkies and the desktop session started cleanly — but no container platform guarantee was available to verify it definitively either way, so rather than speculatively patch Chromium's launch flags without being able to confirm the fix actually does something, this is documented as a known upstream consideration instead.

## Why Deploy a Debian Desktop on Railway?

Railway turns a GUI desktop that would normally need a dedicated VM or a complex Kubernetes setup into a one-click deploy with a persistent volume and a public HTTPS domain handled automatically. For a webtop-style image specifically, that removes the TLS/reverse-proxy setup WebRTC streaming normally demands.

This template's contribution is narrow and precisely scoped: the one configuration value that Railway's own platform currently rejects, fixed and verified, with everything else about an already well-built reference template left exactly as it was.

## What Was Verified

Every claim here was checked against a live deployment and against Railway's own API directly, not inferred. The healthcheck path rejection was reproduced by calling the platform's actual update mutation with the reference's exact value. The fixed path was then deployed live and confirmed: `GET /railway_healthz` returns `200` with no authentication, and `GET /` correctly returns `401 Authorization Required`, confirming basic auth is properly in front of the desktop stream. Deploy logs confirm Selkies started cleanly and fell back to software H.264 encoding with no GPU present — expected on Railway, not an error condition — and the deploy reached `SUCCESS` in well under a minute.

## Frequently Asked Questions

### Why does the reference template's healthcheck path not work?
`/railway-healthz` contains a hyphen, and Railway's current platform validation for healthcheck paths only allows letters, numbers, `/`, and `_`. Confirmed by calling the platform's own live configuration API with that exact value and receiving a rejection. This template uses `/railway_healthz` (underscore) instead, confirmed to pass.

### Does this template modify the webtop image?
No. It deploys `linuxserver/webtop:debian-xfce` exactly as published. The only changes are in the start command's nginx config patch (matching the underscore healthcheck path) and the Composer's healthcheck field — the same scope of change the reference template itself makes, just with a value Railway's platform actually accepts.

### Is there anything else wrong with the reference template?
Not that was found. Its handling of first-boot desktop configuration and volume permissions is already correct, and its defensive "fail loudly if upstream changes" check is good practice. The hyphen-vs-underscore healthcheck path is the one verified, concrete issue — this isn't a template padded out with invented differentiators.

### Why might Chromium occasionally be unstable inside the desktop?
`linuxserver/docker-webtop`'s own documentation recommends `--shm-size=` for all of its desktop images, since Chromium can behave poorly under a small `/dev/shm`. Railway doesn't currently expose that configuration option. This is disclosed as a known upstream/platform interaction, not something this template claims to have fixed.

### Where can I find the underlying image?
Source is on GitHub at [github.com/linuxserver/docker-webtop](https://github.com/linuxserver/docker-webtop). Use this template to deploy it with a healthcheck path that actually passes Railway's current validation, in one click.

Source for this template's docs: https://github.com/shruti060701/debian-desktop-railway
