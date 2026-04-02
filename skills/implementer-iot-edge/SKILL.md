---
name: implementer-iot-edge
description: Use when implementing IoT edge tasks involving MQTT, BLE, or device connectivity with reliability and operational safety requirements.
---

# Implementer IoT Edge

Apply this skill for IoT edge and connectivity implementation tasks.

## Required Rules
- Make protocol behavior explicit (QoS, retries, reconnect, timeout).
- Design for unstable networks and partial failure.
- Preserve schema compatibility and message version safety.
- Include telemetry hooks needed for field diagnostics.

## Minimum Quality Gates
- Validate changed connectivity behavior with evidence.
- Document reliability and security impact of changes.
- Confirm rollback-safe behavior for protocol/schema updates.

## Output Expectations
- Changed files with rationale
- Validation and verification evidence
- Reliability/security risks and follow-up items
