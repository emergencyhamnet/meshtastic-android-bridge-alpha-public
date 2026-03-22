# Known Issues - Alpha 0.2.0-alpha.1

Release date: 2026-03-20
Status: Active for current alpha build

## How To Use This File

- Add one issue per section using the template below.
- Keep status current so operators can quickly see risk.
- Move fixed items to a future release note once verified.

## Status Values

- Open: Confirmed issue, not fixed in this build.
- Monitoring: Mitigation in place, needs more field data.
- Fixed-next: Patch prepared for the next release.
- Closed: Verified resolved in a released build.

## Active Issues

### ALPHA-001 - Private channel UI allows unsupported values

- Status: Open
- Severity: Medium
- Area: Settings UI / Channel routing
- First seen: 2026-03-21
- Affected build: 0.2.0-alpha.1
- Summary: Channel selectors allow Private 8 to Private 10, but Meshtastic supports private channels 1 to 7.
- Impact: Operators can save unsupported channel values, causing unreliable or invalid routing behavior.
- Workaround: Configure only Private 1 to Private 7.
- Tester warning text: Do not use Private 8, Private 9, or Private 10 in this build.
- Planned fix: Clamp all channel selectors and persisted values to 1..7 in next release.

### ALPHA-002 - Duplicate channel assignment can make target resolution ambiguous

- Status: Monitoring
- Severity: Medium
- Area: Caller mapping / Channel-bound routing
- First seen: 2026-03-21
- Affected build: 0.2.0-alpha.1
- Summary: If multiple preferred-caller rows use the same private channel, single-target resolution may not behave as operators expect.
- Impact: Replies may route to a different configured caller than intended in edge cases.
- Workaround: Keep private channel assignments unique per caller row.
- Tester warning text: Avoid assigning the same private channel to more than one caller.
- Planned fix: Add save-time warning and/or validation for duplicate channel assignments in next release.

### ALPHA-003 - Preferred caller count is not hard-limited

- Status: Open
- Severity: Medium
- Area: Settings UI / Governance guardrail
- First seen: 2026-03-21
- Affected build: 0.2.0-alpha.1
- Summary: The intended governance model is not enforced yet: shared-group callers should be capped at 10, while dedicated 1:1 channel callers should be managed separately.
- Impact: Without this split-policy guardrail, operators may configure caller rosters in ways that exceed intended group coordination limits.
- Workaround: Keep shared-group callers at 10 or fewer, and use dedicated private channels for individual contacts.
- Tester warning text: Keep shared-group caller count at 10 or fewer. Dedicated 1:1 channel callers are allowed outside that shared-group cap.
- Planned fix: Add save-time validation to enforce max 10 shared-group callers, enforce one caller per dedicated private channel, and show a clear UI error when policy is exceeded.

### ALPHA-004 - Routing mode is single-active (no simultaneous hybrid mode)

- Status: Monitoring
- Severity: Medium
- Area: Routing model / UX
- First seen: 2026-03-21
- Affected build: 0.2.0-alpha.1
- Summary: Shared and channel-bound profiles can both be configured, but only one mode is active at runtime.
- Impact: Operators expecting shared-group and dedicated-channel behavior at the same time may route traffic differently than intended.
- Workaround: Treat routing modes as presets and switch mode explicitly in User Settings; confirm active mode in System Settings before operations.
- Tester warning text: Only one routing mode is active at a time. Last saved mode is the active mode.
- Validation ask for testers: Report which hybrid behavior is needed most (shared default with dedicated overrides, or dedicated default with shared fallback) and any ambiguity seen in the field.
- Planned fix: Evaluate a true hybrid routing mode with explicit precedence rules based on tester feedback.

## New Issue Template

### ALPHA-XXX - Short title

- Status: Open
- Severity: Low | Medium | High
- Area: Component or feature
- First seen: YYYY-MM-DD
- Affected build: 0.2.0-alpha.1
- Summary: What is wrong.
- Impact: What can go wrong for operators/testers.
- Workaround: Safe operating guidance for current build.
- Tester warning text: Optional copy-ready warning.
- Planned fix: Candidate fix scope or target release.
