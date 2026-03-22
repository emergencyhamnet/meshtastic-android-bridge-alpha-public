# Alpha Upgrade Notes: 0.2.0-alpha.1

Release date: 2026-03-20

## Summary

This alpha tightens SMS <-> Meshtastic routing behavior, loop prevention, and channel-bound safety.

## Key Changes

- Unified outbound echo suppression using shared dedupe state.
- Reduced short-term suppression windows to 5 seconds to avoid over-filtering.
- Improved notification dedupe ordering and added repeated-content throttling.
- Enforced channel-bound policy to block public-channel (`ch=0`) mesh-to-SMS forwarding.
- Restored and improved private-channel reply routing using conversation mapping.
- Added channel-aware conversation mapping to improve reply accuracy in multi-channel use.
- Added Android manifest and receiver compatibility updates for modern platform/lint behavior.

## Upgrade Impact

- Public channel traffic is now excluded from SMS forwarding when channel-bound routing is active.
- Duplicate/repeat notification content is throttled to reduce repeated SMS sends.
- Existing settings and preferred caller mappings remain compatible.

## Validation Checklist

1. Send SMS to mesh and confirm private/shared channel delivery.
2. Reply from mesh private channel and confirm return to correct phone.
3. Verify public channel messages do not bridge to SMS.
4. Confirm repeated status notifications do not re-send within the throttle period.

## Known Limitations

- Gradle/AGP deprecation warnings are still present and intentionally deferred to avoid compatibility regression.
- Multi-caller and multi-channel behavior should continue field validation.
- Preferred-caller governance guardrails are not yet enforced in UI validation:
	- Intended policy is max 10 shared-group callers.
	- Dedicated 1:1 channel callers are separate from the shared-group cap.
- Routing mode is single-active in this build: Shared and Channel-bound profiles can both be configured, but only the last saved mode is active at runtime.

## Operator Warning For This Build

- Private channel selection currently allows values beyond Meshtastic's supported private channel range.
- Until the next release patch is applied, use Private 1 through Private 7 only.
- Do not configure Private 8, Private 9, or Private 10.
- Keep one unique private channel per caller row when possible.
- Keep shared-group caller count at 10 or fewer.
- Only one routing mode is active at a time; last saved mode is active.

## Known Issues Tracking

- See `docs/14_known_issues_alpha_0.2.0.md` for active issues and workarounds.
- Tester feedback requested for hybrid routing maturity: shared-default with dedicated overrides vs dedicated-default with shared fallback.
