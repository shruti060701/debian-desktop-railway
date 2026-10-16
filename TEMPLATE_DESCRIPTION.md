Debian Desktop — a real XFCE desktop in your browser

## Composer description field (paste this in)

A real XFCE Debian desktop in the browser, not just a web terminal. Healthcheck that actually passes.

## Longer version (for context)

A full XFCE desktop — file manager, terminal, Chromium — streamed to your browser over WebRTC via Selkies, from the official `linuxserver/webtop:debian-xfce` image (GPL-3.0). This template fixes the one real, confirmed issue in the official reference: its `/railway-healthz` healthcheck path contains a hyphen, a character Railway's own platform validation currently rejects (confirmed directly against the live service-update API). This template uses `/railway_healthz` instead, verified live to return 200 with zero auth required.

## Why this description
Checked the marketplace field first. The reference's own description — "A real XFCE Debian desktop in the browser, not just a web terminal." — is already strong: it correctly differentiates this product from the `debian` web-terminal-only template in the same marketplace. Kept it verbatim rather than reinventing it, and appended the one genuine, verified differentiator: a healthcheck path that actually passes Railway's current validation, which the reference's own hyphenated path does not.
