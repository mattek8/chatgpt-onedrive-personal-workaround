# OneDrive Personal readback reference behavior — 16 September 2026

This note records a repeated live observation on the tested ChatGPT Microsoft SharePoint/OneDrive connector with a **OneDrive Personal / consumer Microsoft Account (MSA)**.

It supplements the 5 September native baseline and the 16 September expanded primitive validation.

## Observation

The same disposable OneDrive item was read repeatedly through two different references:

1. its canonical Microsoft Graph item target, derived from stable `drive_id` + `item_id`;
2. its normal `onedrive.live.com` browser/web URL.

Across successive checks, the **canonical Graph item target continued to read successfully**, including raw-file materialization, while the browser URL path repeatedly returned:

```text
HTTP 403
accessDenied
serviceReadOnly
Database Is Read Only
```

The item itself, its ID and its parent drive remained unchanged. Native personal-drive discovery/listing continued to work, and the canonical Graph target still returned the expected bytes.

## Interpretation

Current evidence does **not** support treating this as a OneDrive Personal drive becoming genuinely read-only.

The narrower observed failure is associated with the connector's **browser-URL resolution/readback path**, while the exact Graph item path remains healthy.

This distinction matters operationally because a human-facing OneDrive browser URL and a machine-facing stable item identity are not equally reliable references for document-store automation.

## Document-store rule

For persistent OneDrive items, store and prefer:

- `drive_id`;
- `item_id`;
- canonical Graph item target derived from those identifiers;
- human-readable path as descriptive metadata.

Treat the normal browser URL as **navigational metadata for humans**, not as the primary technical identity used for automated readback or verification.

### Preferred verification order

1. use the exact Graph target for post-write readback;
2. if that exact read fails transiently, retry the same exact target;
3. keep persistence `PENDING_VERIFY` until readback succeeds;
4. do not duplicate a successful write merely because a browser-URL fetch failed;
5. invoke the seed fallback only when a required native primitive or the exact Graph path itself is persistently unavailable.

## Impact on `serviceReadOnly`

`serviceReadOnly` should no longer be documented only as a generic intermittent readback failure.

On the tested connector version it has been repeatedly reproducible on the browser-URL resolution path while the canonical Graph exact path for the same item remains functional.

The failure is still provider/connector-version specific and may change in future releases. Revalidation should therefore distinguish:

- native personal-drive capability health;
- exact Graph item readback health;
- browser/sharing URL resolution health.

## Routing remains unchanged

The preferred workflow remains:

1. native OneDrive Personal primitives;
2. canonical Graph exact target for stable technical identity and verification;
3. direct Graph/upload-session paths where appropriate;
4. legacy seed fallback only for actual native/exact-path regressions;
5. manual fallback last.

This observation strengthens the native-first design rather than weakening it.