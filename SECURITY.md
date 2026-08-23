# Security and sensitive data

This repository documents a connector compatibility workaround. The behavior observed during testing did **not** demonstrate an OAuth bypass, privilege escalation or unauthorized access.

## Do not publish secrets in issues

When reporting a reproduction or failure, remove or redact:

- Microsoft access/refresh tokens;
- temporary Graph/SharePoint copy-monitor URLs;
- URLs containing `tempauth`, signatures or other authorization query parameters;
- personal OneDrive `drive_id` values when they are not necessary;
- private item IDs, file names or project names;
- private document hashes if publishing them would fingerprint confidential material;
- personal account information.

A sanitized error code/message, connector action name, date, account type and reproduction step are usually sufficient.

## Actual vulnerabilities

If you discover behavior that crosses an authorization boundary, exposes another user's data, bypasses user consent, or otherwise constitutes a genuine security vulnerability, do not publish exploitation details in this repository first.

Use the appropriate OpenAI and/or Microsoft security reporting process instead. This repository can discuss the compatibility behavior publicly after sensitive details have been handled responsibly.

## Scope of this project

Issues about changed connector behavior, MSA compatibility, reproducibility, safer variants of the workaround, integrity verification and migration behavior are in scope.

Requests for credential bypasses, unauthorized access or techniques to defeat security controls are not.
