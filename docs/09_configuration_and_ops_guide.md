# 09 - Configuration and Operations Guide

This guide helps decide safe and practical runtime configuration for the Android bridge PoC.

It focuses on:
- Valid inbound SMS text formats.
- Private channel behavior and public channel restrictions.
- Operational monitoring and expected logs.
- Emergency-mode options.

## 1) Current configuration surface (today)

Current behavior is controlled mostly in code constants and parser rules:
- `BridgeConfig.kt` for policy defaults.
- `BridgeMessageParser.kt` for accepted inbound SMS message format.
- `EmpV1FrameEncoder.kt` for outbound EMP V1 frame content.
- `BridgeForegroundService.kt` for queue, retry, and retry backoff.
- `MeshtasticOutboundTransport.kt` for USB transport write behavior.

Current important defaults:
- Allowlist: empty (all senders allowed).
- Inbound prefix: `EHN`.
- Inbound direction required on SMS ingress: `S2M`.
- Inbound ID length: 8 to 12 alphanumeric chars.
- Inbound TTL token format: `T<number>`.
- Inbound payload max: 280 chars.
- Outbound EMP destination default: `MESH01`.
- Outbound flags default: `CHAN=private`.
- Retry policy: 4 max attempts, exponential backoff from 1.5s to 30s max.

## 2) Valid inbound SMS texts (what should pass)

Inbound SMS parsing currently expects this high-level shape:

`EHN|S2M|<MSGID>|T<TTL>|<PAYLOAD>`

Validation checklist:
1. Prefix must be `EHN`.
2. Direction must be `S2M`.
3. Message ID must be 8 to 12 alphanumeric chars.
4. TTL token must be like `T30`.
5. TTL value must be in configured range.
6. Payload must be non-empty and within max length.
7. Message ID must not be a recent duplicate.

### Good examples

- `EHN|S2M|A1B2C3D4|T30|Need pickup at north gate`
- `EHN|S2M|ZX91PLM2|T15|Medic team moving to checkpoint 2`

### Reject examples

- `EHN|M2S|A1B2C3D4|T30|Wrong direction for SMS ingress`
- `EHN|S2M|AB12|T30|ID too short`
- `EHN|S2M|A1B2C3D4|30|Missing T in TTL`
- `EHN|S2M|A1B2C3D4|T999|TTL out of configured range`

## 2A) Trusted sender setup flow (plain SMS without markers)

Your assumption is valid as a desired operating mode:
- A known sender sends normal SMS text with no protocol markers.
- Bridge recognizes sender as trusted.
- Bridge forwards message onto Meshtastic private channel.

Important distinction:
1. Current PoC behavior does not do this yet by default.
2. Current parser still expects `EHN|S2M|...` format on inbound SMS.
3. Current outbound transport builds EMP V1 frame bytes for USB write.

### Recommended configuration model for trusted senders

Use two inbound modes:
1. `STRUCTURED_SMS`:
- Require full `EHN|S2M|...` message from sender.

2. `TRUSTED_PLAINTEXT_SMS`:
- Allow plain text from allowlisted sender.
- Bridge generates internal metadata (`MSGID`, `TTL`, timestamps) automatically.

### What the Meshtastic text should look like

If the operator goal is human readability on radio clients, use a render policy of:
1. Preserve payload text exactly.
2. Add optional compact origin prefix only if needed for operations.

Current build behavior:
1. The bridge prefixes forwarded text with origin metadata using this format:
- `"<origin>" - "<message>"`
2. For SMS ingress, origin is normalized phone number.
3. For RCS/notification ingress, origin is sender identity from notification metadata (name or number depending on what Android exposes).

Examples:
1. Plain mirror (recommended for readability):
- SMS in: `Need pickup at north gate`
- Meshtastic out: `Need pickup at north gate`

2. Minimal source context (optional):
- SMS in: `Need pickup at north gate`
- Meshtastic out: `SMS:+15551234567 Need pickup at north gate`

### How reply should work

Reply handling should use a short-lived conversation map:
1. On SMS -> Mesh forward, store mapping:
- `mesh_thread_id` or `(destination, messageId)`
- originating phone number
- timestamp/expiry

2. On Mesh -> SMS reply, route using that mapping:
- If operator replies in the same thread/window, bridge sends SMS back to the mapped sender.
- If mapping expired or ambiguous, require explicit target format (for example `@+15551234567 message`).

3. Keep loop prevention on both directions:
- dedupe by message ID
- direction gate
- replay window

### Recommended first profile

For field usability, start with:
1. Inbound mode: `TRUSTED_PLAINTEXT_SMS` for allowlisted numbers only.
2. Outbound channel mode: `PRIVATE_ONLY`.
3. Meshtastic render policy: plain mirror text.
4. Emergency mode OFF by default.
5. Emergency override can temporarily allow public fallback if explicitly enabled.

## 3) Private channel features and public-channel policy

### Recommended baseline policy

Use private-only egress by default:
1. Keep outbound `FLAGS` as `CHAN=private`.
2. Keep destination set to a private route target (for example `MESH01` now, replace with real route identity later).
3. Reject any runtime config attempt that sets `CHAN=public` unless explicit emergency mode is active.

### Recommended policy levels

1. `PRIVATE_ONLY`:
- Force `CHAN=private`.
- Block sends configured as public.
- Best default for normal operations.

2. `PRIVATE_PREFERRED`:
- Default to private.
- Permit public only with explicit override.
- Useful for controlled field trials.

3. `PUBLIC_ALLOWED`:
- Allow private or public.
- Not recommended for routine operations.

### Suggested implementation rule

When building EMP frame flags:
1. If mode is `PRIVATE_ONLY` and requested channel is not private, reject and log policy violation.
2. If mode is `PRIVATE_PREFERRED`, allow override only if operator enabled override in settings.
3. If mode is `PUBLIC_ALLOWED`, permit requested flags.

## 4) Monitoring operation (what to watch)

Primary live logs:
- `BridgePoC` tag.
- `AndroidRuntime` for crashes.

Useful events to monitor:
1. Service lifecycle:
- `Bridge service started`
- `Bridge service stopped`

2. SMS ingress:
- `SMS received from ...`
- `SMS rejected: ...`
- `Flagged SMS accepted for S2M pipeline ...`

3. Queue behavior:
- `S2M queued ...`
- `S2M delivered ...`
- `S2M retry scheduled ...`
- `S2M dropped after retries ...`

4. USB and transport:
- USB attach and permission logs.
- `S2M transport USB write OK ...`
- Transport failure reasons (no device, no endpoint, transfer error).

### Operational acceptance checks

A healthy send path should show:
1. Accepted SMS parse.
2. Queue enqueue.
3. Transport write success.
4. Delivered log without retries.

A controlled failure test should show:
1. Queue enqueue.
2. Transport failure.
3. Retry scheduling with increasing delay.
4. Eventual drop after max attempts (if persistent failure).

## 4A) Contact identity and logging levels

This section defines the intended operator-visible identity behavior and logging levels.

### Identity policy (name vs number)

Use this strict rule set:
1. Security identity: always phone number (normalized number string).
2. Display identity: contact name if available, otherwise number.
3. Routing and allowlist checks: never based on name.

Recommended display format in logs/UI:
1. If contact name exists: `Name <+15551234567>`
2. If no name: `<+15551234567>`

### Logging levels

Define three levels:

1. `AUDIT_MIN` (non-disableable, always on)
- Required for operational accountability.
- Must always emit at least:
	- timestamp
	- direction (`SMS->MESH` or `MESH->SMS`)
	- from identity
	- to identity
	- message ID
	- result (`accepted`, `rejected`, `queued`, `delivered`, `failed`)

2. `STANDARD` (default operator mode)
- Includes everything in `AUDIT_MIN`.
- Adds concise reason/status fields (validation reject reason, retry count, queue depth, transport status).
- Message body excluded by default.

3. `DEBUG` (temporary troubleshooting mode)
- Includes everything in `STANDARD`.
- Adds full parser details, payload snippets/full payload, USB endpoint details, and timing metrics.
- Should be time-limited and clearly marked as high-detail mode.

### Non-disableable minimum log requirement

Accepted policy (as requested):
1. A minimum message record must always exist and cannot be turned off.
2. At minimum each message event must include:
- `timestamp`
- `from`
- `to`
- `messageId`
- `direction`
- `result`

### Suggested audit event examples

1. `AUDIT_MIN` example:
- `2026-03-18T22:04:31Z SMS->MESH from=Alice <+15551234567> to=MESH01 msgId=AB12CD34 result=queued`

2. `STANDARD` example:
- `2026-03-18T22:04:31Z SMS->MESH from=Alice <+15551234567> to=MESH01 msgId=AB12CD34 result=retry_scheduled retry=2 delayMs=3000`

3. `DEBUG` example:
- `2026-03-18T22:04:31Z SMS->MESH from=Alice <+15551234567> to=MESH01 msgId=AB12CD34 result=usb_write_ok iface=1 ep=0x02 bytes=96 payload='Need pickup at north gate'`

### Recommended privacy defaults

1. `AUDIT_MIN`: no payload content.
2. `STANDARD`: payload omitted or redacted.
3. `DEBUG`: payload allowed only when operator explicitly enables debug mode.

## 5) Emergency option design (recommended)

### Goal

Allow urgent traffic handling while preserving normal private-only safety.

### Proposed emergency mode

Add a runtime toggle with a strict timeout:
1. `Emergency mode OFF` by default.
2. When ON, valid for a limited window (for example 15 minutes).
3. Auto-revert to OFF when timer expires.

### Proposed emergency behavior

When emergency mode is ON:
1. Add or enforce `PRIO=3` in `FLAGS`.
2. Add `ACKREQ` in `FLAGS` for critical messages.
3. Optionally permit `CHAN=public` if private route is unavailable and operator explicitly enabled public fallback.
4. Emit clear audit logs whenever emergency routing is used.

### Suggested emergency payload conventions

Operator-friendly options:
1. Prefix payload with `EMERG:` for human visibility.
2. Include compact structured hints such as location or callback.

Example:
- `EHN|S2M|R4SCUE77|T10|EMERG: Need evacuation at trail marker 12`

## 6) Practical decision guide

Use this if you want a safe first field profile:
1. Allowlist enabled with known tester numbers only.
2. Channel mode: `PRIVATE_ONLY`.
3. Destination: private route target for your node plan.
4. Retry attempts: 4.
5. Backoff: 1.5s initial, 30s max.
6. Emergency mode: available but OFF by default, timed auto-expiry.

## 7) Next implementation steps

1. Move hardcoded config values into persisted settings.
2. Add settings UI for:
- allowlist
- destination
- channel mode
- emergency toggle and timeout
- retry/backoff knobs
3. Enforce channel mode policy in outbound encoder path.
4. Add operation status panel in app UI (last send result, queue depth, last error).

## 8) Practical testing runbook (current build)

Use this quick runbook to validate the fastest-path implementation now in the app.

### Preconditions

1. App installed and bridge service can start.
2. Phone connected to test Meshtastic node over USB OTG.
3. Live logs open with `BridgePoC` filter.

### Test A: Trusted plain-text SMS ingress

Goal:
- Validate inbound mode accepts plain SMS from trusted sender and queues to mesh.

Steps:
1. Ensure allowlist is empty (open mode) or includes your sender number.
2. Send plain SMS to bridge phone, for example:
- `Need pickup at trailhead`
3. Observe logs for:
- audit line with `direction=SMS_TO_MESH ... result=queued`
- `S2M queued ...`
- transport write result (`S2M transport USB write OK ...`) or retry lines
- audit line with `result=delivered` (or `failed_after_retries`)

Expected result:
1. Message accepted without `EHN|S2M|...` markers in trusted plain-text mode.
2. Message is sent on private policy flags unless emergency/public fallback rules apply.

### Test B: Structured SMS mode fallback

Goal:
- Confirm strict parser still works when configured for structured mode.

Steps:
1. Set inbound mode to `STRUCTURED_SMS`.
2. Send:
- `EHN|S2M|A1B2C3D4|T30|Structured test message`
3. Send invalid sample:
- `hello world`

Expected result:
1. Valid structured text is queued/delivered.
2. Invalid text is rejected with parse reason in audit logs.

### Test C: Retry/backoff behavior

Goal:
- Confirm queue retries and drop behavior.

Steps:
1. Disconnect USB node (or simulate transport failure condition).
2. Send trusted SMS.
3. Observe logs:
- `S2M retry scheduled ...`
- increasing delays
- eventual `S2M dropped after retries ...`
- audit result `failed_after_retries`

### Test D: Mesh reply mapping to SMS

Goal:
- Confirm reply-to-sender routing model.

Steps:
1. Send SMS and ensure it reaches `delivered` state.
2. Trigger mesh-reply path in app integration (or via internal action hook) using original `messageId`.
3. Observe logs for `direction=MESH_TO_SMS ... result=delivered`.

Expected result:
1. Reply is routed to originating phone identity from conversation map.

Live mesh command support (current build):
1. The bridge keeps a persistent USB serial listener active while service is running and enqueues SMS replies automatically.
2. Supported reply forms from mesh text:
- `shortname - <text>` (primary operator format)
- `REPLY <messageId> <text>` / `M2S <messageId> <text>` / `@ <messageId> <text>` (explicit messageId)
3. Deprecated command forms:
- `REPLY <shortname> <text>` and `REPLY <text>` are ignored and logged with a deprecation hint.
4. Shared-channel mode fallback:
- If inbound mesh text has no command and there is exactly one preferred caller configured in shared mode, bridge routes plain text to that caller.
5. Channel-bound mode fallback:
- If inbound mesh text has no command and exactly one preferred caller is mapped to the inbound private channel, bridge routes plain text to that caller.
- If no channel-bound match exists and shared mode has exactly one caller, bridge falls back to that shared single caller.
6. Ambiguity guard:
- If there are zero or multiple matching callers after fallback checks, plain text is ignored and logged.
7. If no valid mapping exists, bridge logs `rejected_no_conversation_mapping` and does not send SMS.
8. If mesh text does not match command format, bridge logs `M2S mesh text ignored ...` for troubleshooting.

### Test D1: RCS-originated identity behavior

Goal:
- Clarify when RCS ingress can be replied to automatically via SMS.

Current behavior in this build:
1. RCS notifications are ingested through Notification Listener.
2. The app attempts to extract a phone identity in this order:
- `tel:` URI fields from notification people/person metadata.
- phone-like number found in conversation title or notification text.
- fallback to display name/title when no number is present.
3. Automatic Mesh->SMS reply requires a phone-like identity (normalized digits, typically full international format).

Operational implications:
1. If notification metadata includes a real number, automatic reply mapping can work.
2. If RCS notification exposes only a contact name, automatic reply mapping cannot guarantee SMS routing.
3. In that case, configure preferred caller mapping (`shortname -> phone (+ optional RCS name)`) and/or use manual reply target input.

Preferred caller mapping (new):
1. Configure in app UI field `Preferred callers`.
2. Each row has `short name`, `phone`, optional `RCS display name`, optional `private channel`.
3. Matching is case-insensitive against short name and RCS display name.
4. Mapped number is used for conversation reply routing when sender is not phone-like.
5. Reply routing mode:
- `Shared channel`: one caller list, optional single-caller plain-text fallback.
- `Channel-bound`: separate caller list, optional per-caller channel mapping (Private 1..10).
6. Shared channel is explicitly configured in User Settings (`Private 1..Private 10`), default `Private 1`.
7. In shared mode, SMS->mesh always uses configured shared channel.
8. In channel-bound mode, SMS->mesh uses mapped caller channel; if mapping is missing, bridge falls back to configured shared channel.
9. Channel selection no longer infers from destination label (for example `MESH01`).
10. Use full international numbers for best SMS reliability.

Known platform limitation:
1. Android public APIs do not provide a universal, reliable way for third-party apps to send true RCS chat replies.
2. This bridge sends replies via SMS telephony, not native RCS send APIs.

### Minimum acceptance criteria

1. Non-disableable audit fields present in logs:
- timestamp
- from
- to
- messageId
- direction
- result
2. Trusted plain-text ingress works for known sender.
3. Private-only policy enforced unless emergency/public fallback is explicitly active.
4. RCS ingress handling documented as best-effort identity extraction with SMS-only reply egress.
