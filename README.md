# ChatGPT ↔ OneDrive Personal connector compatibility research

> **Independent community research — not an official OpenAI or Microsoft project.**
>
> Original seed-based workaround discovered, tested and first documented in August 2026 by **[@mattek8](https://github.com/mattek8)** with ChatGPT (OpenAI).

This repository started as a reproducible workaround for making **OneDrive Personal / Microsoft Account (MSA)** writable from ChatGPT when the Microsoft SharePoint/OneDrive connector exposed only a partial set of useful Graph-backed operations.

## Current status — 5 September 2026

**The connector surface changed materially. Native OneDrive Personal operations are now available and were validated end-to-end on the same MSA account used for the original workaround research.**

The seed workflow is therefore no longer the preferred path on the currently tested connector. It remains documented as a validated fallback and as a historical record of the August 2026 connector behavior.

Observed native OneDrive Personal primitives on 5 September 2026:

| Operation | Current tested result on MSA | Connector primitive |
| --- | --- | --- |
| Read known file | ✅ | `fetch` |
| List folder children | ✅ | `list_drive_item_children` |
| Search personal drive files/folders | ✅ | `search_drive_items` |
| Create folder | ✅ | `create_drive_folder` |
| Upload new file | ✅ | `upload_drive_item_content` |
| Upload PDF/binary | ✅ | `upload_drive_item_content` |
| Replace existing file contents | ✅ | `update_file_exact` |
| Rename / move | ✅ | `move_drive_item` |
| Delete file to recycle bin | ✅ | `delete_drive_item` |
| Delete folder to recycle bin | ✅ | `delete_drive_item` |
| Copy file/folder | ✅ | `copy_item` |

A disposable TXT workflow and an independent PDF workflow were both completed successfully. See [`NATIVE_VALIDATION_2026-09-05.md`](NATIVE_VALIDATION_2026-09-05.md).

### Important documentation mismatch

At the time of this revalidation, OpenAI's public SharePoint help documentation still stated that **Personal OneDrive accounts are not supported by the SharePoint app**. The live connector available to the tested ChatGPT account nevertheless exposed and successfully executed OneDrive-personal-specific create/upload/list/search/move/delete operations.

This repository therefore reports **observed connector behavior**, not an official product support guarantee. The capability may be a rollout, compatibility path, or documentation lag and should be re-tested on other accounts and connector versions.

## What changed from August 2026

### 23 August 2026

On the tested MSA connection:

- SharePoint-oriented native folder creation failed for MSA;
- native SharePoint delete was unavailable for the personal drive path;
- Microsoft Search/general discovery was unavailable or incomplete;
- exact Graph-backed copy, item resolution, overwrite, rename and move operations did work.

A seed-based compatibility layer was built from those working primitives:

1. copy a folder seed to create a directory;
2. copy or reuse a one-byte file seed to create a file;
3. resolve the new item through its Graph path;
4. overwrite the placeholder with the real bytes;
5. verify the destination through readback/SHA-256;
6. use `_trash` as a soft-delete fallback.

That method was independently reproduced twice and then used for two real document-store migrations: **13/13 pre-existing files matched byte-for-byte after OneDrive readback**.

The complete original README is preserved unchanged in [`LEGACY_SEED_WORKAROUND.md`](LEGACY_SEED_WORKAROUND.md), and the August validation record remains in [`VALIDATION.md`](VALIDATION.md).

### 5 September 2026

The same connector family now exposed dedicated personal-drive actions including:

- `search_drive_items`;
- `list_drive_item_children`;
- `create_drive_folder`;
- `upload_drive_item_content`;
- `move_drive_item`;
- `delete_drive_item`.

These were not merely visible in the connector schema: they were exercised against the connected OneDrive Personal account.

## Recommended workflow now

Use capability-based routing rather than forcing the workaround:

1. **Native OneDrive Personal primitives** — preferred on the current connector.
2. **Direct Microsoft Graph / upload session** — preferred when available and appropriate, especially for large files.
3. **Seed-based fallback** — use only if native personal-drive actions are absent, fail, or regress.
4. **Manual intervention** — last resort.

### Minimal native create/write/read/delete cycle

1. Create the destination folder with `create_drive_folder`.
2. Upload the file with `upload_drive_item_content`.
3. Persist the returned `drive_id` / `item_id` or canonical item URL when useful.
4. Verify the write with `fetch` or raw download.
5. Use `list_drive_item_children` for deterministic immediate folder inspection.
6. Use `search_drive_items` for discovery, allowing for indexing latency.
7. Rename or move with `move_drive_item`.
8. Delete with `delete_drive_item` when deletion is explicitly intended.

For important persistent originals or migrations, a positive upload response should still not be treated as a substitute for content verification. Prefer destination readback and SHA-256 comparison when raw bytes are available.

## Search / discovery caveat

The new `search_drive_items` primitive successfully searched the personal OneDrive and returned canonical IDs for both files and folders.

However, during the 5 September revalidation, a file created only seconds earlier did **not** immediately appear in text search even though:

- the upload had succeeded;
- direct `fetch` by returned item ID worked;
- `list_drive_item_children` showed the file immediately.

Treat OneDrive search as potentially eventually consistent. For just-created content, prefer the IDs returned by the write operation or direct folder listing.

## Native revalidation summary

The 5 September 2026 live test covered two disposable workflows.

### TXT cycle

- native folder creation;
- native upload of a 104-byte TXT file;
- immediate folder listing;
- exact content readback with a unique marker;
- native rename while preserving the same item ID;
- native file deletion;
- verification that the folder was empty;
- native folder deletion.

### PDF cycle

- native folder creation;
- native upload of a valid 624-byte `application/pdf` file;
- readback through the connector with the expected PDF text extracted;
- native file deletion;
- native folder deletion.

No seed item was used in either native validation cycle.

See [`NATIVE_VALIDATION_2026-09-05.md`](NATIVE_VALIDATION_2026-09-05.md) for the sanitized evidence record.

## The seed workaround remains useful as a fallback

The workaround is retained because connector capabilities can differ across accounts, plans, rollouts and future versions. If the native personal-drive primitives are missing or stop working, the August 2026 technique is already validated.

Minimal fallback layout:

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

The legacy method and its edge cases — asynchronous `HTTP 202` copy, temporary `404`, no-op path resolution, copied timestamps and soft-delete — are documented in [`LEGACY_SEED_WORKAROUND.md`](LEGACY_SEED_WORKAROUND.md).

## Permissions and security framing

This research does **not** demonstrate a security bypass.

The tested Microsoft consent already granted ChatGPT file access sufficient to read, create, update and delete accessible OneDrive files. The original limitation was in the connector surface exposed to the MSA account, not in an attempt to obtain additional privileges.

The seed workaround and the newer native path both operate within the permissions granted to the connected application and remain subject to ChatGPT confirmation controls where configured.

Never publish:

- personal drive IDs unless necessary;
- private item IDs or filenames;
- temporary copy-monitor URLs;
- URLs containing temporary authentication parameters or tokens.

If an actual security vulnerability is discovered, use the appropriate vendor security-reporting channel rather than publishing exploit details. See [`SECURITY.md`](SECURITY.md).

## Large files

Different layers have different limits:

- OneDrive/SharePoint storage supports much larger files than the one-shot Graph content API.
- Microsoft Graph one-shot `PUT .../content` supports up to 250 MB per call.
- Microsoft Graph exposes resumable upload sessions for larger transfers.
- ChatGPT sync/indexing limits are separate from storage/upload limits.

The original seed workflow was not stress-tested near the one-shot ceiling. The new native `upload_drive_item_content` action states that large files use an upload session automatically, but this repository has **not yet independently stress-tested large-file behavior** on the 5 September connector version.

## Revalidation checklist for another account/version

Before relying on the connector for important data:

1. confirm the account type is OneDrive Personal / MSA;
2. create a disposable folder natively;
3. upload a small disposable file natively;
4. list the folder and read the file back;
5. search for the item and note any indexing delay;
6. rename or move it;
7. delete the file and folder;
8. repeat once with a binary/PDF file;
9. only then treat native MSA support as verified for that environment;
10. fall back to the seed workflow if the native primitives are unavailable or regress.

## Files in this repository

- [`README.md`](README.md) — current status and recommended workflow.
- [`NATIVE_VALIDATION_2026-09-05.md`](NATIVE_VALIDATION_2026-09-05.md) — native MSA connector revalidation.
- [`LEGACY_SEED_WORKAROUND.md`](LEGACY_SEED_WORKAROUND.md) — preserved August 2026 README describing the seed workaround in full.
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
- Microsoft Graph — createUploadSession: https://learn.microsoft.com/en-us/graph/api/driveitem-createuploadsession
- Microsoft Support — Restrictions and limitations in OneDrive and SharePoint: https://support.microsoft.com/en-us/onedrive/restrictions-and-limitations-in-onedrive-and-sharepoint

## License

Copyright © 2026 **mattek8**.

Except where otherwise noted, the original documentation and validation material is available under **CC BY 4.0**. Third-party product names, documentation and trademarks remain the property of their respective owners.
