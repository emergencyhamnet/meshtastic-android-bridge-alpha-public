# Alpha Release Notes (2026-03-19)

Tag: `alpha-2026-03-19`

## Included artifacts

1. `releases/alpha_2026-03-19/meshtastic-bridge-alpha-2026-03-19.apk`
2. `releases/alpha_2026-03-19/SHA256SUMS.txt`

## Highlights

1. Stable foreground bridge service with persistent USB serial listener.
2. SMS/RCS ingress to mesh delivery with dedupe and audit logging.
3. Mesh-to-SMS reply routing with preferred caller model.
4. Shared-channel and channel-bound routing modes.
5. Explicit shared channel setting for predictable send behavior.
6. Operator reply support:
   1. preferred: `shortname - message`
   2. plain text fallback when exactly one effective target is resolvable
7. Tester quickstart and smoke test documentation included.

## Known constraints

1. Android APIs do not provide universal third-party RCS send support; outbound is SMS telephony.
2. Deprecated `REPLY <shortname> <message>` convenience format is intentionally ignored.

## Validation baseline

1. Build: successful.
2. Device install: successful on Pixel 8 Pro test target.
