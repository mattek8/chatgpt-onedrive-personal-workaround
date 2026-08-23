# ChatGPT ↔ OneDrive Personal writable workaround

> **Independent community research — not an official OpenAI or Microsoft project.**
>
> Original workaround discovered, tested and first documented in August 2026 by **[@mattek8](https://github.com/mattek8)** with ChatGPT (OpenAI).

This repository documents a reproducible workaround for using **OneDrive Personal / Microsoft Account (MSA)** as a writable document store when ChatGPT's Microsoft SharePoint/OneDrive connector exposes some Microsoft Graph-backed primitives that work on a personal drive, while other convenient connector actions such as native folder creation, search/discovery or delete are unavailable or incompatible with the tested MSA account.

The workaround is intentionally documented as a **compatibility technique**, not as a security vulnerability. It does not bypass OAuth, gain additional permissions, or circumvent ChatGPT confirmation controls. It composes operations that the connected account is already authorized to perform.

## Status

**Validated on 23 August 2026.**

- 2 independent synthetic end-to-end reproductions.
- 2 migrations of already-populated real-world document stores.
- 13/13 pre-existing files verified byte-for-byte after migration.
- File types exercised: TXT, PDF, JPEG, ZIP and generic binary content.
- Folder creation, file creation, overwrite, rename, move, readback and soft-delete exercised.
- OneDrive Personal / consumer MSA account tested.

See [`VALIDATION.md`](VALIDATION.md) for the validation protocol and its limits.

## The short version

On the tested MSA connection, the convenient high-level connector surface was incomplete: some SharePoint-oriented actions did not work for OneDrive Personal. However, exact Microsoft Graph-backed item operations did work.

That makes the following pattern possible:

1. keep one tiny **folder seed** and one tiny **file seed**;
2. copy the folder seed to create a new folder;
3. copy or reuse the file seed to create a new file;
4. resolve the newly copied item through its known Graph path;
5. overwrite the placeholder with the real file bytes;
6. download/read the destination again and verify SHA-256;
7. use an `_trash` folder as a soft-delete fallback when native delete is unavailable.

In other words, the missing high-level `mkdir` / `touch` workflow can be reconstructed from **copy + exact item resolution + overwrite**.

## Prefer direct OneDrive operations when available

This repository is named after the workaround, but **the workaround should not be used when a simpler native path exists**.

Recommended capability-based routing:

1. **Native/direct OneDrive write** exposed by the current environment — preferred.
2. **Microsoft Graph direct upload / upload session** — preferred when available, especially for large files.
3. **Seed-based ChatGPT MSA fallback described here** — use when the connector cannot natively create/upload/delete/search on the personal drive.
4. Manual intervention only when no suitable programmatic primitive exists.

This matters for environments such as Codex or other tools that may have a filesystem, a OneDrive client, or a more complete Graph integration. The seed flow performs multiple operations and is naturally slower than a direct upload.

## Tested connector primitives

The following action names describe the connector surface observed during testing. They are implementation details and may change over time.

| Operation | Tested result on MSA | Working pattern |
| --- | --- | --- |
| Read known file | ✅ | exact item / Graph reference + `fetch` |
| Copy file | ✅ | `copy_item` (`HTTP 202`, asynchronous) |
| Copy folder | ✅ | `copy_item` (`HTTP 202`, asynchronous) |
| Create file | ✅ workaround | file seed → copy/rename → `update_file_exact` |
| Create folder | ✅ workaround | copy folder seed |
| Resolve known Graph path to item ID | ✅ | no-op `move_or_rename_item` |
| Rename | ✅ | `move_or_rename_item` |
| Move | ✅ | `move_or_rename_item` with exact targets |
| Overwrite TXT/PDF/binary | ✅ | `update_file_exact` |
| Readback after write | ✅ | `fetch` / raw download |
| Microsoft Search / general discovery | ❌ on tested MSA surface | persist IDs and known paths instead |
| Native `create_folder` action | ❌ on tested MSA surface | folder seed copy |
| Native SharePoint delete action | ❌ on tested MSA surface | move to `_trash` |

## Bootstrap

A minimal stable layout is:

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

`AI Projects/seed/` is the folder seed. `AI Projects/seed/seed` is a neutral one-byte file.

### One-time setup

1. Connect the Microsoft SharePoint/OneDrive app to the OneDrive Personal account with the permissions required by the workflow.
2. Manually create `AI Projects/seed/` if the connector cannot create the first folder.
3. Put a one-byte file named `seed` inside it.
4. Create `AI Projects/_trash/`, or create it later by copying the folder seed.
5. Obtain the personal drive's `drive_id` from a known/recent item returned by the connector.
6. Resolve the seed folder and persist its stable item reference for future operations.

Do **not** publish personal drive IDs, item IDs, temporary monitor URLs or URLs containing authorization parameters when sharing logs or examples.

## Resolving an item from a known Graph path

Given a known `drive_id`, a path can be represented as:

```text
https://graph.microsoft.com/v1.0/drives/<drive_id>/root:/AI Projects/<path>
```

On the tested connector, calling `move_or_rename_item` against that path while setting `new_name` to the **same current name** behaved as a no-op resolver. The response returned the real item ID and parent metadata.

This was reproduced for both files and folders created through asynchronous copy operations.

## Creating a folder with the seed fallback

1. Copy `AI Projects/seed/` into the desired parent folder using the final folder name.
2. Treat `HTTP 202` as **accepted**, not as proof that the copy is already materialized.
3. Retry/back off until the new path resolves.
4. Resolve it with the no-op rename pattern and persist the returned item ID.
5. The copied directory inherits a one-byte `seed` file. Either transform it into the first real file or move it to `_trash`.

An immediate `404` after `HTTP 202` was observed in testing and later resolved successfully. Do not treat one immediate `404` as a definitive copy failure.

## Creating a file with the seed fallback

There are two useful cases.

### A. A seed already exists in the newly copied folder

1. Rename `seed` to the final filename.
2. Replace its contents with the real bytes using the exact item write primitive (`update_file_exact` on the tested connector).
3. Use the correct MIME type.
4. Download/read the destination again.
5. Compare SHA-256 against the source.

### B. No seed exists in the destination folder

1. Copy `AI Projects/seed/seed` into the destination with the final name.
2. Wait for asynchronous copy materialization.
3. Resolve the new item from its known Graph path.
4. Overwrite it with the real bytes.
5. Read it back and compare SHA-256.

## Integrity verification: do not trust upload success alone

For persistent originals, migrations and other important writes, the recommended invariant is:

```text
source bytes
   ↓ SHA-256 A
write to OneDrive
   ↓
read/download destination bytes
   ↓ SHA-256 B
A == B  → VERIFIED
```

A successful API response proves that an operation was accepted or completed according to that API; it does not replace an independent content-integrity check.

If the environment cannot retrieve the destination bytes, record the weaker verification that was actually performed instead of claiming byte-for-byte equivalence.

## Updating, renaming and moving

For an existing item, prefer the simplest exact operation exposed by the current environment.

In the tested connector:

- `update_file_exact` replaced the complete file contents;
- `move_or_rename_item` handled rename and move;
- item IDs remained stable across moves within the same drive during the tests.

For container formats such as DOCX, PPTX and XLSX, replacing the file means replacing the complete binary package; this technique is not an internal document-format patch mechanism.

## Soft-delete fallback

When native delete is unavailable on the connector surface, move unwanted items to:

```text
AI Projects/_trash/
```

This is deliberately a **soft delete**, not a claim that the item was deleted from OneDrive. The trash folder can be reviewed and emptied manually through OneDrive until a compatible delete primitive is available.

## Large files and important limits

Different layers have different limits.

### OneDrive storage

Microsoft documents a general maximum of **250 GB per individual file** in OneDrive/SharePoint.

### Microsoft Graph one-shot content upload

Microsoft Graph's `PUT .../content` upload/replace API supports files up to **250 MB in one call**. The exact overwrite primitive used in the tested connector is operationally similar to this one-shot model, so the seed fallback should **not** be assumed validated beyond that size.

### Microsoft Graph upload sessions

For larger or less reliable transfers, Graph exposes `createUploadSession`. Microsoft documents sequential resumable ranges, each request under 60 MiB, with intermediate range sizes that are multiples of 320 KiB, and recommends resumable transfers for files larger than 10 MiB.

The tested ChatGPT connector surface did **not** expose an upload-session primitive during this research. Prefer a native/direct integration that does when large files matter.

### ChatGPT SharePoint sync is a separate layer

OpenAI currently documents a **100 MB per-file** maximum for the SharePoint app's sync/indexing path. A file being validly stored in OneDrive does not therefore guarantee that it can be indexed by ChatGPT sync.

The workaround has **not** been stress-tested near the 250 MB one-shot ceiling.

## Validation summary

The method was validated in four stages:

- synthetic reproduction #1: text, real PDF and generic binary write/readback plus copy/move/soft-delete;
- synthetic reproduction #2: clean bootstrap from a folder seed and one-byte file seed, including observed eventual consistency and path resolution;
- real document-store migration #1: 10 pre-existing files;
- real document-store migration #2: 3 pre-existing files.

For the two real migrations, every source file was downloaded, hashed, written to OneDrive, downloaded again from OneDrive and hashed again. **13/13 destination hashes matched their source hashes.**

Private file names, project names, personal OneDrive IDs and document hashes are intentionally not published here. See [`VALIDATION.md`](VALIDATION.md).

## What this is — and what it is not

This is:

- a reproducible compatibility workaround;
- evidence that useful exact Graph-backed item operations can work on the tested OneDrive Personal connection even when parts of the higher-level connector surface do not;
- a way to build a writable document-store workflow while waiting for a cleaner native connector path.

This is **not**:

- an OAuth bypass;
- a privilege escalation;
- evidence of unauthorized access;
- a claim that Microsoft Graph itself cannot create folders or upload files to OneDrive Personal;
- an official or supported OpenAI/Microsoft API contract.

If you discover an actual security vulnerability rather than a connector-compatibility issue, do not post secrets or exploit details in a public issue; use the appropriate vendor security-reporting channel instead. See [`SECURITY.md`](SECURITY.md).

## Reproduction checklist

Before trusting the technique on another account or connector version:

1. check whether a simpler direct/native OneDrive write is now available;
2. obtain a `drive_id` without publishing it;
3. resolve the folder/file seed;
4. create one disposable folder via seed copy;
5. create and overwrite one disposable file;
6. download it and verify SHA-256;
7. create a nested folder;
8. move the file and confirm the same item remains readable;
9. move it to `_trash` and read it again;
10. only then mark that environment as verified.

Connector behavior can change. Please open an issue if a step stops working, including the date, account type and non-sensitive error message — **never tokens or temporary authenticated URLs**.

## Attribution and citation

If this workaround, documentation, or validation methodology is reused or discussed, please credit the original source:

> **mattek8 — `chatgpt-onedrive-personal-workaround` (2026)**  
> https://github.com/mattek8/chatgpt-onedrive-personal-workaround

Machine-readable citation metadata is provided in [`CITATION.cff`](CITATION.cff).

The original documentation and validation material in this repository is licensed under **Creative Commons Attribution 4.0 International (CC BY 4.0)**. See [`LICENSE`](LICENSE).

## References

- Microsoft Graph — Upload or replace the contents of a driveItem: https://learn.microsoft.com/en-us/graph/api/driveitem-put-content?view=graph-rest-1.0
- Microsoft Graph — driveItem: createUploadSession: https://learn.microsoft.com/en-us/graph/api/driveitem-createuploadsession?view=graph-rest-1.0
- Microsoft Support — Restrictions and limitations in OneDrive and SharePoint: https://support.microsoft.com/en-us/onedrive/restrictions-and-limitations-in-onedrive-and-sharepoint
- OpenAI Help Center — SharePoint app in ChatGPT: https://help.openai.com/en/articles/12143177-sharepoint-app-in-chatgpt

## License

Copyright © 2026 **mattek8**.

Except where otherwise noted, the original documentation and validation material is available under **CC BY 4.0**. Third-party product names, documentation and trademarks remain the property of their respective owners.
