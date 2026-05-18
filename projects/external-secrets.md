# External Secrets

Field notes from Se7en's work around `external-secrets/external-secrets`.

## Vault SecretStore validation and KV mount metadata

External Secrets' Vault provider validates a `SecretStore` before individual `ExternalSecret` reads happen. It is tempting to make that validation also prove that the configured Vault KV mount path and KV version are correct, but the boundary is subtle.

Important constraints:

- Vault application tokens are often scoped to business secret paths, not broad system metadata APIs.
- `sys/mounts` lists Vault mounts and commonly requires broader system-level policy than a normal application token should receive.
- `sys/internal/ui/mounts/<path>` is narrower and can work from path capabilities, but it is an internal Vault UI/preflight endpoint and can still be unavailable depending on policy, namespace, mount visibility, or Vault behavior.
- `spec.provider.vault.path` may be omitted. In that mode, the actual Vault mount can be encoded in each `ExternalSecret.remoteRef.key`, so a SecretStore-level validator may not know which mount to check.

A compatibility-preserving validator can only fail when it has positive metadata proving that the configuration is wrong, such as a non-`kv` mount or a KV v1/v2 mismatch. Metadata lookup errors should not automatically make the `SecretStore` NotReady, because that would turn missing metadata permissions into a breaking change for users whose secret reads still work.

Practical rule: keep SecretStore validation focused on credentials/connectivity unless mount metadata can be checked without requiring broader Vault policy. Treat Vault path/version validation as best-effort at SecretStore scope and definitive at actual read time.
