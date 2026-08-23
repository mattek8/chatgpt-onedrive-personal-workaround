# Validation record

This document records the validation evidence behind the workaround without publishing private project data, personal OneDrive identifiers, temporary authenticated URLs, source filenames or document hashes.

## Validation date

23 August 2026.

## Environment

- OneDrive Personal / consumer Microsoft Account (MSA).
- ChatGPT Microsoft SharePoint/OneDrive connector surface available at the time of testing.
- Exact Microsoft Graph-backed item operations exposed through the connector.

Connector action names are implementation details and may change.

## Pass criteria

A write or migration was considered **verified** only when the destination content was independently read/downloaded again after the write.

For real-file migration, the integrity criterion was:

1. download the source file;
2. calculate source SHA-256;
3. create/resolve the OneDrive destination item;
4. write the original bytes;
5. download the destination from OneDrive;
6. calculate destination SHA-256;
7. require source SHA-256 == destination SHA-256.

Metadata equality alone was not considered sufficient.

## Synthetic validation #1

The first exploratory end-to-end run demonstrated that exact item operations on the tested OneDrive Personal account could be composed into a useful writable workflow.

Verified operations included:

- copy an existing item;
- overwrite a copied item with new text content;
- write and reopen a valid PDF;
- write generic binary content;
- resolve folder/item IDs from exact operations;
- copy folders;
- rename and move items;
- perform a logical soft-delete by moving an item into a trash folder;
- read the moved content again.

This run established feasibility but still contained exploratory dependencies created during investigation.

## Synthetic validation #2 — clean reproduction

The procedure was then repeated from a clean folder seed plus a one-byte file seed.

Sequence:

1. copy the folder seed into a new disposable directory;
2. copy/reuse the one-byte file seed as a new file;
3. observe that an immediate path read could temporarily return `404` after asynchronous `HTTP 202` copy acceptance;
4. resolve the same item later through a known Graph path using a no-op rename;
5. overwrite the file with completely new content;
6. read the new content back successfully;
7. create a nested folder from the folder seed;
8. move the file into the nested folder;
9. confirm the item remained readable;
10. create a soft-trash folder;
11. move the file into soft-trash;
12. confirm the same item remained readable after the move.

This second run reduced the chance that the technique only worked because of accidental state left by the exploratory test.

## Real migration validation #1

A previously populated document store was migrated from another cloud provider into a newly constructed OneDrive project structure.

- Pre-existing files: **10**.
- Types represented: ZIP, PDF, JPEG.
- Folder structure created through the seed workflow.
- Every source file was downloaded and SHA-256 hashed.
- Every destination was independently downloaded from OneDrive after write.
- Result: **10/10 SHA-256 matches**.

The source store was intentionally left intact during validation.

## Real migration validation #2

A second independent populated document store was migrated using the same procedure.

- Pre-existing files: **3**.
- Types represented: PDF, TXT.
- Destination structure created through the seed workflow.
- Source and post-write destination bytes were independently hashed.
- Result: **3/3 SHA-256 matches**.

## Aggregate result

Across both real migrations:

- **13 pre-existing files**;
- formats represented: ZIP, PDF, JPEG, TXT;
- **13/13 byte-for-byte SHA-256 matches** after OneDrive readback.

Together with the synthetic tests, the observed workflow covered TXT, PDF, JPEG, ZIP and generic binary data.

## Connector behaviors reproduced

| Behavior | Result |
| --- | --- |
| Exact known-item read | PASS |
| File copy | PASS |
| Folder copy | PASS |
| File creation via seed | PASS |
| Folder creation via seed | PASS |
| No-op path resolution to item ID | PASS |
| Exact overwrite | PASS |
| Rename | PASS |
| Move | PASS |
| Readback after move | PASS |
| Soft-delete via move | PASS |
| Readback after soft-delete | PASS |
| Native connector search/discovery on tested MSA path | NOT AVAILABLE / INCOMPLETE |
| Native connector folder creation on tested MSA path | NOT AVAILABLE |
| Native SharePoint delete action on tested MSA path | NOT AVAILABLE |

## Important negative / untested areas

The validation does **not** establish all possible OneDrive behavior.

Not validated here:

- files larger than 250 MB through the tested one-shot connector write path;
- stress testing close to the 250 MB one-shot ceiling;
- Graph `createUploadSession` through the tested ChatGPT connector (the action was not exposed during testing);
- concurrent writes to the same OneDrive item;
- cross-drive moves;
- preservation semantics for version history;
- native hard-delete on OneDrive Personal through this connector surface;
- completeness or latency of ChatGPT SharePoint sync/indexing after the write;
- every possible Microsoft account, tenant configuration, connector version or ChatGPT plan.

The procedure must therefore be revalidated when the connector surface changes or when used in a materially different environment.

## Privacy of the published evidence

The public repository intentionally omits:

- personal `drive_id` values;
- item IDs;
- project names;
- original private filenames;
- SHA-256 values of private source documents;
- temporary copy-monitor URLs;
- any URL containing temporary authorization parameters.

The absence of those values does not weaken the validation method: the relevant public result is that independently computed source and destination hashes matched for all tested files.

## Revalidation checklist

For a new account or connector version:

1. first test whether a simpler native/direct OneDrive operation is available;
2. if the fallback is still required, resolve the seed folder and seed file;
3. create a disposable folder;
4. create and overwrite a disposable file;
5. read it back and verify SHA-256;
6. create a nested folder;
7. move the file there and read it again;
8. move it into `_trash` and read it again;
9. record the date and non-sensitive connector behavior;
10. only then treat the environment as verified.

If you reproduce or invalidate the procedure, an issue or pull request with the date, account type, connector surface and sanitized error/result is welcome.
