# ChatGPT ↔ OneDrive Personal connector compatibility research

> **Independent community research — not an official OpenAI or Microsoft project.**
>
> Original seed-based workaround discovered, tested and first documented in August 2026 by **[@mattek8](https://github.com/mattek8)** with ChatGPT (OpenAI).

This repository documents the observed behavior of the ChatGPT Microsoft SharePoint/OneDrive connector with **OneDrive Personal / consumer Microsoft Account (MSA)**.

It started as a reproducible seed-based workaround when the connector exposed only a partial writable surface. Since 5 September 2026, the tested connector has exposed dedicated personal-drive primitives that make the workaround unnecessary for normal operation. The seed method remains a validated legacy fallback.

## Current status — 16 September 2026

**Native OneDrive Personal operations remain the preferred and validated workflow on the tested connector.**

The 5 September baseline validated native personal-drive search/listing, direct folder creation, direct upload, exact writes, move/rename, delete and readback. A 16 September follow-up additionally validated bulk operations, version history/restore and effective permission inspection.

Observed live MSA capability matrix:

| Operation | Current tested result | Connector primitive |
| --- | --- | --- |
| Read known file | ✅, transient retry caveat | `fetch` |
| List folder children | ✅ | `list_drive_item_children` |
| Search personal drive | ✅ | `search_drive_items` |
| Create folder | ✅ | `create_drive_folder` |
| Bulk create folders | ✅ | `create_drive_folders_bulk` |
| Upload new file / binary | ✅ | `upload_drive_item_content` |
| Replace existing contents | ✅ | `update_file_exact` |
| Rename / move | ✅ | `move_drive_item` |
| Bulk rename / move | ✅ | `move_drive_items_bulk` |
| Delete to recycle bin | ✅ | `delete_drive_item` |
| Copy file/folder | ✅ | `copy_item` |
| List version history | ✅ | `list_item_versions` |
| Restore prior version | ✅ | `restore_item_version` |
| List effective permissions | ✅ | `list_item_permissions` |
| Anonymous sharing link | Exposed, not automatically tested | `create_drive_item_link` |
| Named-recipient invitation | Exposed, not automatically tested | `invite_item_recipients` |

See:

- [`NATIVE_VALIDATION_2026-09-05.md`](NATIVE_VALIDATION_2026-09-05.md) — original native-MSA revalidation;
- [`NATIVE_VALIDATION_2026-09-16.md`](NATIVE_VALIDATION_2026-09-16.md) — bulk, versioning, permission inspection and transient readback validation.

## Recommended workflow now

Use capability-based routing:

1. **Native OneDrive Personal primitives** — preferred when present and working.
2. **Direct Microsoft Graph / upload session** — use when appropriate, especially in environments with direct Graph access or for specialized large-file workflows.
3. **Seed-based fallback** — legacy compatibility path only when native personal-drive actions are unavailable or persistently regress.
4. **Manual intervention** — last resort.

Do not force the seed workaround when a direct personal-drive primitive exists.

### Minimal native create/write/read/delete cycle

1. Create the destination folder with `create_drive_folder`.
2. Upload with `upload_drive_item_content`.
3. Keep the returned `drive_id` / `item_id` or canonical item URL.
4. Verify the write using exact-item readback.
5. Use `list_drive_item_children` for deterministic immediate folder inspection.
6. Use `search_drive_items` for discovery, allowing for indexing latency.
7. Rename/move with `move_drive_item`, or batch known operations with `move_drive_items_bulk`.
8. Delete with `delete_drive_item` when deletion is explicitly intended.

For important originals and migrations, a successful write response is not equivalent to byte verification. Prefer destination readback plus SHA-256 comparison.

## New native primitives validated on 16 September

### Bulk folder creation

`create_drive_folders_bulk` created two folders in one request on `drive_type: personal`.

The operation is useful for project bootstrap and migration, but it is **not atomic**. Inspect every per-item result before retrying or continuing.

### Bulk move / rename

`move_drive_items_bulk` moved and renamed two files in one request while keeping the same item IDs.

This makes structured reorganizations cheaper in connector round trips, with the same non-atomic caveat.

### Version history / restore

A disposable file was written as 30 bytes, replaced with a 42-byte version, then inspected through `list_item_versions`:

- `2.0` — 42 bytes;
- `1.0` — 30 bytes.

`restore_item_version(1.0)` returned the item to 30 bytes and produced a new `3.0` version. Subsequent raw readback matched the original source byte-for-byte by SHA-256.

Provider version history can therefore be used as an **additional rollback layer** for mutable document-store items and exports. It does not replace Git history, provenance or explicit persistence verification.

### Permission inspection

`list_item_permissions` successfully returned the effective owner permission for the test item.

This is useful for ACL/sharing audits. Sharing mutations are deliberately not part of automatic validation: anonymous links and named-recipient invitations change access to content and should only be executed under explicit user intent.

## Known transient failure: `serviceReadOnly`

The connector has intermittently returned:

```text
HTTP 403
accessDenied
serviceReadOnly
Database Is Read Only
```

on **raw readback**, even while other operations against the same OneDrive Personal account continue to work.

On 16 September the error was reproduced immediately after a successful version restore. A retry a few seconds later against the **same exact item** succeeded, and the returned bytes matched the expected source hash.

Current interpretation: `serviceReadOnly` is a **transient readback/materialization failure mode**, not evidence that the personal drive itself has permanently become read-only.

Recommended handling:

1. keep the exact item reference returned by the write;
2. if raw fetch returns `serviceReadOnly`, leave verification pending;
3. retry the same exact-item read after a short delay;
4. distinguish repeated readback failure from loss of create/upload/move/delete primitives;
5. only fall back to the seed workflow when a required native primitive is absent or persistently unusable;
6. mark byte persistence `VERIFIED` only after successful destination readback and hash comparison.

## Search / discovery caveat

`search_drive_items` successfully searches the personal OneDrive and returns canonical IDs for files and folders.

During the 5 September test, a file created only seconds earlier was not immediately returned by text search even though upload, exact fetch and folder listing had succeeded.

Treat drive search as potentially **eventually consistent** for just-created items. Prefer returned item IDs or direct folder listing immediately after a write.

## What changed from August 2026

### 23 August 2026

The tested MSA connector did not yet expose a complete native create/upload/search/delete workflow. A seed compatibility layer was therefore built from available exact Graph-backed operations:

1. copy a folder seed to create a directory;
2. copy or reuse a one-byte file seed to create a file;
3. resolve the copied item;
4. overwrite the placeholder with the real bytes;
5. verify destination readback/SHA-256;
6. use `_trash` as a soft-delete fallback.

The method was independently reproduced twice and used for two real document-store migrations: **13/13 pre-existing files matched byte-for-byte after OneDrive readback**.

The complete original method is preserved in [`LEGACY_SEED_WORKAROUND.md`](LEGACY_SEED_WORKAROUND.md), with its original validation record in [`VALIDATION.md`](VALIDATION.md).

### 5 September 2026

Dedicated personal-drive search/list/create/upload/move/delete primitives appeared and were exercised successfully end-to-end. Native-first became the documented default.

### 16 September 2026

Bulk folder creation, bulk move/rename, version history/restore and permission inspection were validated on MSA. A transient `serviceReadOnly` raw-read failure was also reproduced and then cleared on retry against the same item.

## Document-store implications

For projects using OneDrive as an external document store:

- bulk create/move can reduce round trips during bootstrap and migrations;
- provider versioning can provide rollback for mutable files and generated exports;
- permission inspection can be used for access audits;
- sharing changes should remain explicit, user-authorized operations;
- readback retries should be part of verification before treating `serviceReadOnly` as a persistent regression.

The architecture remains unchanged: **GitHub/state store for state and provenance; OneDrive for persistent originals/binaries where appropriate.**

## The seed workaround remains useful as fallback

The legacy layout remains compatible:

```text
OneDrive/
└── AI Projects/
    ├── seed/
    │   └── seed
    ├── _trash/
    └── <project-name>/
        ├── originals/
        └── exports/
```

`seed/` and `_trash/` can be retained harmlessly, but they are no longer required by the normal native path.

The original edge cases — asynchronous `HTTP 202` copy, temporary `404`, copied timestamps, resolver behavior and soft delete — remain documented in [`LEGACY_SEED_WORKAROUND.md`](LEGACY_SEED_WORKAROUND.md).

## Permissions and security framing

This research does **not** demonstrate a security bypass.

The tested connector operates inside access granted to the connected Microsoft account and remains subject to ChatGPT confirmation controls. Connector confirmation state and Microsoft authorization are separate layers.

Earlier tests observed a ChatGPT "always allow" confirmation choice for connector changes. Native writes remained operational afterwards, but `serviceReadOnly` later recurred on readback, so current evidence does not establish the UI confirmation setting as the cause or cure for that failure.

Do not publish private drive IDs, personal item IDs, temporary download/upload URLs, account identifiers or tokens.

## Large files

OneDrive storage limits, Graph upload limits and ChatGPT sync/indexing limits are separate concerns.

The connector contract for `upload_drive_item_content` states that large files reuse upload-session handling, but this repository has **not yet independently stress-tested the size threshold or very-large-file behavior** on MSA.

For important large files, continue to use post-write readback/hash verification when technically possible.

## Public documentation mismatch

OpenAI's public SharePoint help article was checked again on **16 September 2026**. It still states that **Personal OneDrive accounts are not supported by the SharePoint app**, while documenting live SharePoint actions such as folder creation, file upload/update and sharing-link management for supported environments.

The tested live connector nevertheless continues to expose and execute personal-drive-specific MSA primitives.

This repository therefore reports **observed live connector behavior**, not an official guarantee of OneDrive Personal support for every account, plan or rollout.

Reference: https://help.openai.com/en/articles/12143177

## Revalidation checklist

Before relying on the connector for important data on another account/version:

1. confirm that the connected account is OneDrive Personal / MSA;
2. create a disposable folder natively;
3. upload a small disposable file;
4. list and read the file back;
5. test exact overwrite;
6. test rename/move;
7. test delete;
8. optionally test bulk create/move if the workflow will use them;
9. optionally test version history/restore on disposable content;
10. treat `serviceReadOnly` as retryable until repeated checks show otherwise;
11. verify important persistence with destination readback + SHA-256;
12. use the seed fallback only if native primitives are unavailable or persistently fail.

## Files in this repository

- [`README.md`](README.md) — current status and recommended workflow.
- [`NATIVE_VALIDATION_2026-09-05.md`](NATIVE_VALIDATION_2026-09-05.md) — first native MSA connector revalidation.
- [`NATIVE_VALIDATION_2026-09-16.md`](NATIVE_VALIDATION_2026-09-16.md) — expanded primitive validation and transient readback behavior.
- [`LEGACY_SEED_WORKAROUND.md`](LEGACY_SEED_WORKAROUND.md) — preserved August seed workaround.
- [`VALIDATION.md`](VALIDATION.md) — August workaround validation record.
- [`SECURITY.md`](SECURITY.md) — security-reporting guidance.
- [`CITATION.cff`](CITATION.cff) — citation metadata.

## Attribution and citation

If this research, workaround, documentation or validation methodology is reused or discussed, please credit the original source:

> **mattek8 — `chatgpt-onedrive-personal-workaround` (2026)**  
> https://github.com/mattek8/chatgpt-onedrive-personal-workaround

## References

- OpenAI Help Center — SharePoint app and setup in ChatGPT: https://help.openai.com/en/articles/12143177
- Microsoft Graph — Upload or replace driveItem content: https://learn.microsoft.com/en-us/graph/api/driveitem-put-content
- Microsoft Graph — driveItem versions: https://learn.microsoft.com/en-us/graph/api/driveitem-list-versions
- Microsoft Graph — restore driveItem version: https://learn.microsoft.com/en-us/graph/api/driveitemversion-restoreversion
- Microsoft Graph — createUploadSession: https://learn.microsoft.com/en-us/graph/api/driveitem-createuploadsession

## License

Copyright © 2026 **mattek8**.

Except where otherwise noted, the original documentation and validation material is available under **CC BY 4.0**. Third-party product names, documentation and trademarks remain the property of their respective owners.