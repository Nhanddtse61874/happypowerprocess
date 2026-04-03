---
name: implementer-react-native-typescript
description: Use when implementing React Native TypeScript tasks that need stable mobile behavior, maintainable UI boundaries, and release safety.
---

# Implementer React Native/TypeScript

Apply this skill for React Native implementation tasks.

## Required Rules
- Keep component/state boundaries explicit and predictable.
- Handle loading/error/offline states for user-facing flows.
- Account for iOS/Android differences and device limitations.
- Keep implementation aligned to approved acceptance criteria.
- Keep side effects isolated and cleaned up (`useEffect` cleanup, unsubscribe listeners).

## Minimum Quality Gates
- Add or update tests for critical user paths.
- Verify runtime behavior for changed flows.
- Note platform-specific assumptions and risks.

## Output Expectations
- Changed files with rationale
- Test and verification evidence
- Platform risks and follow-up items

## Performance
- Use `useMemo` only for expensive derived values with measured impact.
- Use `useCallback` for stable props passed to memoized child components.
- For `FlatList`, provide stable `keyExtractor` and avoid creating heavy inline closures in hot paths.
- Avoid unnecessary re-renders: do not call `setState` when the next value is equivalent.

## Error Handling
- Ensure a top-level error boundary exists for render-time failures.
- Every async hook/service call must surface failure state (`error`, retry action, user message).
- Never swallow errors silently; log enough context for diagnosis.

## TypeScript Rules
- Avoid `any`; use exact types or `unknown` with type guards.
- Define interfaces for all device models, MQTT payloads, and navigation params.
- Use strict null checks - always handle `undefined` / `null` explicitly.

## State Management
- Local state -> `useState`
- Shared app state -> `useContext`
- Introduce Redux/Zustand only when context-based composition is no longer maintainable.

## Conditional Add-Ons
- If task includes MQTT/BLE/device connectivity, also apply `skills/implementer-iot-edge/SKILL.md`.
- If app uses global navigation guards or crash analytics, include verification for those integrations.

## Anti-Patterns to Avoid
- Inline large objects/functions in list item render paths.
- Business logic embedded directly in presentational components.
- Async effects without cancellation/unsubscribe on unmount.
- UI that shows spinner forever without explicit error/empty/offline fallback.

## Verification Matrix
- Type safety: run TypeScript checks for changed scope.
- Tests: run relevant unit/integration/e2e tests for touched flows.
- Runtime: verify changed behavior on at least one real platform target (Android or iOS).
- Regression: confirm loading/error/offline states are still reachable and correctly rendered.

