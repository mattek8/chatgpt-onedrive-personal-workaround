# Expanded OneDrive Personal connector validation — 16 September 2026

This document records a sanitized live validation of newly exposed Microsoft SharePoint/OneDrive connector primitives against a **OneDrive Personal / consumer Microsoft Account (MSA)**.

It intentionally omits personal drive IDs, item IDs, account identifiers, private filenames, temporary URLs and authorization parameters.

## Context

The 5 September 2026 baseline had already validated the native OneDrive Personal workflow end-to-end: personal-drive search/listing, direct folder creation, direct upload, exact replacement writes, rename/move, readback and delete. The older seed-based workflow remained only as a validated legacy fallback.

Subsequent connector builds exposed additional primitives, especially bulk operations, version history/restore and sharing/permission inspection. This test determines which of those primitives are merely present in the schema and which actually work on the tested MSA account.

## Environment

- ChatGPT Plus individual account.
- Microsoft OneDrive Personal / consumer MSA.
- Microsoft SharePoint/OneDrive connector as exposed by ChatGPT on 16 September 2026.
- Existing Microsoft consent already granted; ChatGPT mutation confirmations configured to allow connector changes without repeated prompts.
- Disposable test content under an existing AI project area.

The test does **not** claim general availability for every account, plan, region or rollout.

## Results

| Primitive / capability | Result | Notes |
| --- | --- | --- |
| `create_drive_folders_bulk` | ✅ PASS | Two folders created in one batch on `drive_type: personal`. |
| `move_drive_items_bulk` | ✅ PASS | Two files moved and renamed in one batch; item IDs remained stable. |
| `update_file_exact` | ✅ PASS | Test file changed from 30 to 42 bytes and generated a new version. |
| `list_item_versions` | ✅ PASS | Version history returned 2.0 (42 B) and 1.0 (30 B). |
| `restore_item_version` | ✅ PASS | Restoring 1.0 returned the file to 30 B; a new 3.0 version was then visible. |
| Post-restore byte readback | ✅ PASS after retry | Raw fetch first hit transient `serviceReadOnly`, then succeeded; restored bytes matched the original SHA-256 exactly. |
| `list_item_permissions` | ✅ PASS | Effective permission inspection returned the expected owner role on the disposable item. |
| Anyone-with-link creation | ⚪ EXPOSED / NOT TESTED | The connector exposes the primitive, but creating anonymous access is intentionally not part of an automatic validation run. |
| Named-recipient invitation | ⚪ EXPOSED / NOT TESTED | The connector exposes the primitive, but recipient sharing requires explicit user intent and a chosen recipient. |

## Bulk folder creation

`create_drive_folders_bulk` created two sibling folders in one request under the disposable test root.

Result: **PASS**.

Both per-item results reported success and identified the parent as a personal OneDrive drive.

Operational consequence: project bootstrap and migration workflows can create several known folders with fewer connector round trips. The operation is explicitly **not atomic**, so callers must inspect the per-item result list before retrying or continuing.

## Bulk move / rename

Two disposable text files were uploaded into the first test folder. A single `move_drive_items_bulk` request then:

- moved both files to the second folder;
- renamed both files;
- preserved each item's stable OneDrive item ID.

Result: **PASS** for both entries.

Operational consequence: reorganizations and migrations can use bulk moves, but must retain the same non-atomic / inspect-before-retry discipline.

## Version history and restore

A 30-byte text file was replaced with a 42-byte version using `update_file_exact`.

`list_item_versions` then returned:

- version `2.0` — 42 bytes;
- version `1.0` — 30 bytes.

`restore_item_version` was called for `1.0`.

Result: **PASS**.

The returned item metadata immediately reported 30 bytes again. A second version-history query showed:

- version `3.0` — 30 bytes (the restore result);
- version `2.0` — 42 bytes;
- version `1.0` — 30 bytes.

A later raw readback returned the original v1 text, and its SHA-256 matched the pre-upload source exactly.

Operational consequence: OneDrive provider-level version history can be used as an additional rollback mechanism for mutable document-store items. It does **not** replace GitHub state history, provenance records or byte-level verification for important originals.

## Permission inspection

`list_item_permissions` was executed against the disposable file.

Result: **PASS**.

The effective permission list contained the expected owner role.

Operational consequence: a document-store workflow can audit current ACL/sharing state before or after an explicitly authorized sharing change.

This validation deliberately did not create anonymous links or invite named recipients. Those actions change who can access content and should only be executed when the user explicitly requests the corresponding sharing operation.

## `serviceReadOnly` transient failure reproduced

During the post-restore verification, the first raw `fetch` failed with:

- HTTP 403;
- `accessDenied`;
- inner error `serviceReadOnly`;
- message `Database Is Read Only`.

Other Graph-backed management primitives had just succeeded against the same personal drive, including exact write, version listing and version restore.

A retry a few seconds later against the **same exact item** succeeded and returned the restored file bytes. SHA-256 matched the original source.

This is now a reproduced transient failure mode rather than evidence that the OneDrive itself has become read-only.

### Recommended handling

For important readback verification:

1. keep the exact drive/item reference returned by the successful write;
2. if raw fetch returns `serviceReadOnly`, treat the result as **not yet verified**, not as a failed write;
3. retry the exact-item read after a short delay;
4. only invoke the legacy seed path if the required native primitive itself is unavailable or persistently failing across repeated checks;
5. mark byte persistence `VERIFIED` only after successful destination readback and hash comparison.

## Consent / confirmation observation

Earlier tests had shown ChatGPT mutation confirmations and an "always allow" choice for connector changes. Native writes have remained operational after that change, but `serviceReadOnly` has still recurred on readback.

Therefore the current evidence does **not** support treating the confirmation setting as the root cause or cure for `serviceReadOnly`. UI confirmation state and provider/materialization failures should be tracked separately.

## Document-store implications

The additional primitives improve the document-store workflow without changing its architecture:

- **bulk create/move** — useful for bootstrap, migrations and reorganizations;
- **version history/restore** — useful provider-level rollback for mutable items and exports;
- **permission listing** — useful for ACL/sharing audits;
- **sharing primitives** — potentially useful, but only under explicit user authorization;
- **readback retry** — now part of the verification procedure for transient `serviceReadOnly` failures.

GitHub remains canonical for project state, decisions, metadata, document indexes and provenance. OneDrive remains the persistent store for originals/binaries where appropriate.

## Public documentation mismatch

OpenAI's public SharePoint help article was checked again on 16 September 2026 and still stated that **personal OneDrive accounts are not supported by the SharePoint app**, while also documenting live SharePoint actions such as folder creation, file upload/update and sharing-link management for supported environments.

Reference: https://help.openai.com/en/articles/12143177

The live connector behavior recorded here therefore remains empirical compatibility research, not an official product-support guarantee.

## Updated capability classification

### Validated native MSA workflow

- search/listing;
- create folder;
- bulk create folders;
- direct file upload;
- exact full-content write;
- move/rename;
- bulk move/rename;
- delete to recycle bin;
- readback (with transient retry caveat);
- version history;
- version restore;
- effective permission listing.

### Exposed but not yet end-to-end validated here

- anonymous sharing-link creation;
- named-recipient invitations;
- very-large-file automatic upload-session behavior;
- cross-drive moves (explicitly unsupported by the current move primitive contract).

## Routing decision

The documented workflow remains:

1. **native OneDrive Personal primitives first**;
2. **direct Microsoft Graph / upload session** when appropriate and available;
3. **legacy seed-based fallback** only when native MSA capabilities are absent or persistently regress;
4. manual fallback only as a last resort.

The 16 September test strengthens the native-first recommendation; it does not restore the seed workflow to primary status.