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
