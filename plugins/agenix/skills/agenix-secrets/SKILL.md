---
name: agenix-secrets
description: Use when working with agenix-encrypted secrets: reading, writing, adding, updating, or rekeying .age files, or editing secrets.nix.
version: 1.0.0
---

# agenix Secret Management

agenix encrypts NixOS secrets using `age`. Each `.age` file is encrypted for one or more SSH public key recipients declared in `secrets.nix`. The agenix CLI handles decryption; `age` itself handles encryption.

---

## Reading a Secret

Decrypt a secret to stdout:

```bash
cd /path/to/secrets-dir   # must contain secrets.nix
agenix -d path/to/secret.age -i <path/to/identity-key>
```

Must run from the directory containing `secrets.nix` — agenix looks up the file's registered keys there even for decryption.

---

## Writing or Updating a Secret

`agenix -e` handles piped input automatically — when STDIN is not interactive, agenix sets `EDITOR` to `cp /dev/stdin`, so piping the secret value works directly:

```bash
cd /path/to/secrets-dir   # must contain secrets.nix
printf 'YOUR_SECRET_VALUE\n' | agenix -e path/to/secret.age -i <path/to/identity-key>
```

- Use `printf` (not `echo`) to control trailing newlines precisely
- Must run from the directory containing `secrets.nix` — agenix reads recipient keys from there automatically
- No need to look up or pass recipient keys manually

### Verify immediately

Always decrypt right after writing to confirm the file is not empty and the value is correct:

```bash
agenix -d path/to/secret.age -i <path/to/identity-key>
```

If this outputs nothing or errors, the write failed — check identity key and secrets.nix.

---

## Rekeying All Secrets

Rekey when recipient keys in `secrets.nix` have changed (new host added, key rotated, etc.). Rekeying re-encrypts every `.age` file for the current set of recipients.

**Must run from the directory containing `secrets.nix`:**

```bash
cd /path/to/secrets-dir
agenix -r -i <path/to/identity-key>
```

- The identity key passed with `-i` must already be a current recipient of the secrets being rekeyed — a newly generated key not yet listed in `secrets.nix` will cause a decryption error
- All `.age` files in the directory are re-encrypted
- Files will always differ from before (age uses random nonces)
- Commit the rekeyed files to version control

---

## Key Facts

| Thing | Value |
|---|---|
| Identity key | The agenix SSH identity key — pass with `-i <path/to/identity-key>` |
| Recipient keys | Declared per-secret in `secrets.nix` under `publicKeys` |
| Safe write method | `printf '...' \| agenix -e FILE -i KEY` (pipes work; agenix sets EDITOR to `cp /dev/stdin`) |
| After writing | Verify with `agenix -d` immediately |

---

## Common Mistakes

**Forgetting to verify** → a failed encrypt leaves a zero-byte or corrupt file that will break service startup silently.

**Running `agenix -r` from the wrong directory** → rekey must be run from the directory containing `secrets.nix`, not a subdirectory or parent.

