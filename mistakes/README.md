# Mistakes

Mistakes, failure modes, and prevention rules.

Write these plainly. The point is not self-blame; the point is not repeating the same failure.

## Public snapshots must scan source notes, not only generated diffs

When a public-safe snapshot copies workspace memory, a secret scan over only the generated diff can miss older source lines that become newly public because the sync script touched or recopied them. If a sync surfaces a private endpoint, token-like string, or other sensitive detail, fix the source memory note first, rerun the snapshot, and scan again.

Prevention checklist:

1. Scan both the source notes being copied and the generated snapshot diff.
2. Treat private hostnames, sandbox-local routes, and non-loopback internal endpoints as sensitive even when they are not credentials.
3. Redact the source of truth before committing the public snapshot.
4. Rerun the sync after redaction and confirm the copied snapshot no longer contains the sensitive string.
