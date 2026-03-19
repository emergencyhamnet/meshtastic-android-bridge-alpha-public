# Alpha Tester Smoke Test Plan

This plan is designed for quick alpha validation in 15-25 minutes.

## Preconditions

1. App installed and permissions granted.
2. Notification access granted.
3. USB Meshtastic node connected.
4. Bridge service started.

## Test 1: Shared mode single caller baseline

Setup:
1. User Settings -> Reply routing mode: Shared Channel.
2. Shared channel: Private 1.
3. Preferred callers: exactly one row (for example `sw`, phone, optional RCS name).

Steps:
1. Send SMS/RCS from mapped caller to phone.
2. Confirm SMS/RCS -> mesh deliver in activity log and logcat.
3. Send mesh reply as plain text.
4. Send mesh reply as `sw - test message`.

Expected:
1. Both mesh reply forms deliver to mapped phone.
2. S2M logs show `ch=1` (or selected shared channel).

## Test 2: Channel-bound per-caller routing

Setup:
1. User Settings -> Reply routing mode: Channel-bound.
2. Keep shared channel configured (fallback).
3. Add two channel-bound callers on different channels (for example Private 1 and Private 2).

Steps:
1. Send inbound SMS/RCS from caller A and caller B.
2. Confirm S2M logs show correct channel per caller mapping.
3. Send mesh plain text from channel 1 and channel 2.

Expected:
1. Plain text from each channel routes to its mapped caller when single match exists.
2. If channel has no mapping, fallback behavior is logged clearly.

## Test 3: Ambiguity and guardrails

Steps:
1. Configure multiple callers such that a plain-text reply is ambiguous.
2. Send mesh plain text without shortname.

Expected:
1. Bridge ignores ambiguous reply.
2. Log contains reason explaining the drop.

## Test 4: RCS name mapping

Setup:
1. Ensure preferred caller includes RCS display name.

Steps:
1. Send RCS message where notification sender shows display name.

Expected:
1. Bridge resolves display name to configured caller/phone.
2. Correct channel selection appears in S2M log.

## Pass criteria

1. No crashes.
2. Bridge service remains stable during tests.
3. All expected deliveries occur with correct channel behavior.
4. Drop cases are explicit and understandable in logs.