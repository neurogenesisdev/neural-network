# Source material — being consumed, not current

These documents are inputs to the build-spec refactor. **They are not current
documentation.** The PRD in particular contains at least one specification that
was later rejected (§7.3, DNS over TLS — see ADR-0005) and files Secure Boot and
TPM2 under "Deferred" when both were applied on 2026-08-15.

Delete this directory once `../spec/` supersedes it.

## Redacted 2026-08-22

These files were passed through `../collect/redact.sh` before being committed.
The LUKS container UUID and the NVMe serial became `<FSUUID_nvme0n1p2>` and
`<NVME_SERIAL>`. The procedures are unchanged; only identifiers were replaced.

This is closer to correct than the originals were. A rebuild produces a **new**
LUKS UUID, so the literal value was never a reusable instruction — it was a
record of one particular build.
