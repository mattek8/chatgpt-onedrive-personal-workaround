# Native OneDrive Personal connector validation — 5 September 2026

This document records a sanitized revalidation of the ChatGPT Microsoft SharePoint/OneDrive connector against a **OneDrive Personal / consumer Microsoft Account (MSA)**.

It intentionally omits personal `drive_id` values, item IDs, private filenames, temporary URLs and authorization parameters.

## Why this revalidation was run

The repository originally documented a seed-based compatibility workaround because, on 23 August 2026, the tested connector could use some exact Microsoft Graph-backed item operations on OneDrive Personal but did not expose a complete native MSA workflow for folder creation, search/discovery and deletion.

On 5 September 2026, the connector schema exposed new personal-drive-specific actions. A fresh live test was therefore run to determine whether those actions were merely present in the schema or actually functional on the same OneDrive Personal account used during the original research.

## Environment

- ChatGPT Plus individual account.
- Microsoft OneDrive Personal / consumer MSA.
- Existing Microsoft consent with full file access already granted before this revalidation.
- Microsoft SharePoint/OneDrive connector as exposed by ChatGPT on 5 September 2026.

No OAuth scope change was intentionally made as part of this test.

## Newly observed personal-drive primitives

The connector exposed personal OneDrive-specific actions including:

- `search_drive_items`;
- `list_drive_item_children`;
- `create_drive_folder`;
- `create_drive_folders_bulk`;
- `upload_drive_item_content`;
- `move_drive_item`;
- `move_drive_items_bulk`;
- `delete_drive_item`.

Existing exact-item operations such as `fetch`, `update_file_exact` and `copy_item` remained available.

Action names are implementation details and may change.

## Discovery test

`search_drive_items` was executed against the signed-in personal drive.

Result:

- PASS for existing OneDrive Personal content;
- file and folder results were returned;
- canonical Graph item references, `drive_id`, `item_id` and parent path metadata were returned.

This materially improves the original August state, where general discovery on the tested MSA connector surface was unavailable or incomplete.

## Native TXT end-to-end cycle

A disposable folder was created directly under an existing test area using `create_drive_folder`.

### 1. Native folder creation

Result: **PASS**.

The returned metadata identified the item as a folder with zero children and provided its stable item ID immediately.

### 2. Native file upload

A local 104-byte plain-text file containing a unique validation marker was uploaded directly into the new folder using `upload_drive_item_content`.

Result: **PASS**.

The returned metadata reported:

- the expected filename;
- `text/plain` MIME type;
- 104-byte size;
- a new stable item ID;
- the correct parent folder.

No seed copy or placeholder overwrite was involved.

### 3. Immediate folder listing

`list_drive_item_children` was called on the new folder.

Result: **PASS**.

The just-uploaded file was immediately visible in the deterministic folder listing.

### 4. Readback

`fetch` was called using the exact item reference returned by the upload.

Result: **PASS**.

The connector returned the exact validation text and unique marker expected from the uploaded source file.

### 5. Search latency observation

`search_drive_items` was queried for the newly created filename only seconds after upload.

Result: **NO RESULT YET**.

This did not indicate a failed upload because:

- direct readback had already succeeded;
- folder listing showed the file;
- the exact item ID was valid.

Interpretation: OneDrive drive search should be treated as potentially **eventually consistent** for newly created content. Immediately after writes, prefer the returned item ID or `list_drive_item_children` over text search.

### 6. Native rename

The file was renamed in place with `move_drive_item`.

Result: **PASS**.

The item ID remained unchanged while the filename changed.

### 7. Native file deletion

The renamed file was removed with `delete_drive_item`.

Result: **PASS**.

A subsequent folder listing returned no children.

### 8. Native folder deletion

The now-empty disposable folder was removed with `delete_drive_item`.

Result: **PASS**.

## Native PDF end-to-end cycle

A second independent disposable folder was created natively.

A valid 624-byte PDF containing a unique text marker was then uploaded directly with `upload_drive_item_content` using MIME type `application/pdf`.

### Upload result

Result: **PASS**.

The connector returned:

- expected filename;
- `application/pdf` MIME type;
- 624-byte size;
- correct parent folder;
- stable item ID.

### PDF readback

`fetch` was called on the uploaded PDF.

Result: **PASS**.

The connector parsed the PDF and returned the expected embedded validation text.

This confirms that the new native upload path is not limited to plain text.

### Cleanup

The PDF and its disposable parent folder were both removed with `delete_drive_item`.

Result: **PASS** for both operations.

## Capability matrix after revalidation

| Capability | 23 Aug 2026 connector surface | 5 Sep 2026 connector surface |
| --- | --- | --- |
| Read known item | PASS | PASS |
| Copy file/folder | PASS | PASS |
| Native personal-drive search | unavailable/incomplete | PASS |
| Native deterministic folder listing | unavailable as personal-drive primitive | PASS |
| Native folder creation on MSA | unavailable through tested high-level path | PASS |
| Native new-file upload on MSA | unavailable through tested high-level path | PASS |
| Native PDF/binary upload | workaround required | PASS |
| Exact overwrite | PASS | PASS |
| Rename/move | PASS through exact Graph-backed operation | PASS through dedicated personal-drive operation |
| Native delete on MSA | unavailable through tested SharePoint path | PASS |
| Seed workaround required | YES for full writable workflow | NO on current tested connector |

## Documentation mismatch observed

At the time of this validation, OpenAI's public SharePoint help documentation still stated that **Personal OneDrive accounts are not supported by the SharePoint app**.

The live connector available to the tested account nevertheless exposed dedicated personal-drive primitives and successfully executed native search, folder creation, upload, listing, rename/move and delete against OneDrive Personal.

This record therefore documents **empirical connector behavior**, not an official guarantee that OneDrive Personal is generally supported for every account, plan or rollout.

## Conclusion

**Native OneDrive Personal write support is operational on the tested connector as of 5 September 2026.**

The August seed-based technique remains valid as a fallback, but it should no longer be the default workflow when the dedicated personal-drive primitives are present and functioning.

Recommended routing:

1. use native personal-drive actions first;
2. verify important writes through readback and SHA-256 when raw bytes are available;
3. allow for search indexing latency after recent writes;
4. retain the seed workflow only as a compatibility fallback if native primitives are absent or regress.

## Scope limits

This revalidation did **not** establish:

- behavior on every MSA account, region, plan or connector rollout;
- large-file stress behavior;
- exact threshold at which `upload_drive_item_content` switches to resumable upload sessions;
- cross-drive move behavior;
- all shared-item and shortcut edge cases;
- completeness/latency guarantees for `search_drive_items`;
- ChatGPT sync/indexing support for OneDrive Personal.

Those remain separate questions from the native live-action workflow validated here.
