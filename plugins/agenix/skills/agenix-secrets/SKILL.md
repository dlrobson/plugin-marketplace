---
name: agenix-secrets
description: This skill should be used when reading, writing, adding, updating, removing, renaming, or rekeying agenix-encrypted .age secrets, or editing the secrets.nix recipient declarations.
version: 1.0.0
---

# agenix Secret Management

agenix encrypts NixOS secrets using `age`. Each `.age` file is encrypted for one or more SSH public key recipients declared in `secrets.nix`. The agenix CLI handles decryption; `age` itself handles encryption.

**Every agenix command below must be run from the directory containing `secrets.nix`** — agenix reads recipient keys from there, even for decryption.

---

## secrets.nix Syntax

Each `.age` file gets an entry listing its recipient public keys:

```nix
let
  user1 = "ssh-ed25519 AAAA...";
  host1 = "ssh-ed25519 AAAA...";
in
{
  "secret-name.age".publicKeys = [ user1 host1 ];
}
```

- Add a host/user as a recipient of a secret by adding its public key to that secret's `publicKeys` list, then rekey (see below) so the `.age` file is actually re-encrypted for the new key

---

## Adding a New Secret

1. Add an entry for it in `secrets.nix` with the intended recipients (see above) — agenix looks up recipients from this file, so the entry must exist before encrypting
2. Create and encrypt it the same way as updating an existing secret (below)
3. Verify by decrypting

---

## Reading a Secret

Decrypt a secret to stdout:

```bash
agenix -d path/to/secret.age -i <path/to/identity-key>
```

---

## Writing or Updating a Secret

`agenix -e` handles piped input automatically — when STDIN is not interactive, agenix sets `EDITOR` to `cp /dev/stdin`, so piping the secret value works directly:

```bash
printf 'YOUR_SECRET_VALUE\n' | agenix -e path/to/secret.age -i <path/to/identity-key>
```

- Use `printf` (not `echo`) to control trailing newlines precisely
- No need to look up or pass recipient keys manually — agenix reads them from `secrets.nix`
- For values containing single quotes, `$`, backticks, or other shell-special characters, write the value to a temp file first and pipe with `cat tempfile | agenix -e FILE -i KEY` rather than embedding it in a quoted `printf` argument

### Verify immediately

Always decrypt right after writing to confirm the file is not empty and the value is correct:

```bash
agenix -d path/to/secret.age -i <path/to/identity-key>
```

If this outputs nothing or errors, the write failed — check identity key and secrets.nix.

---

## Removing or Renaming a Secret

**Removing:** delete both the `.age` file and its `secrets.nix` entry. No rekey needed.

**Renaming:** rename the `.age` file (`git mv old-name.age new-name.age`) and update the corresponding key in `secrets.nix` to match. No rekey needed — the ciphertext is unchanged, only the filename and its `secrets.nix` reference move.

---

## Rekeying: Which Method to Use

Default to rekeying only the specific secret(s) whose recipients changed. Only rekey *all* secrets when recipients changed for every secret, or when explicitly asked to rekey everything — a full `-r` touches every `.age` file and produces unnecessary diff noise for unaffected secrets (age uses random nonces, so every rekeyed file changes even if plaintext and recipients didn't).

---

## Rekeying a Single Secret (default choice)

`agenix -r` always rekeys every `.age` file — there is no `-r FILE` option. But `agenix -e` re-encrypts unconditionally (skipping its usual "no changes, skip" check) when `EDITOR` is set to `:`, which is the exact mechanism `-r` uses internally per file. This enables rekeying a single file:

```bash
EDITOR=: agenix -e path/to/secret.age -i <path/to/identity-key>
```

- Does not open an editor and does not change the plaintext — only re-encrypts against the recipients currently listed in `secrets.nix` for that file
- Leaves every other `.age` file untouched (verified: byte-identical before/after)
- Useful for code review noise: after adding/rotating one recipient, rekey only the affected secret(s) instead of the whole directory
- To rekey several specific secrets, loop this command over each file — still avoids touching unrelated secrets
- Verify afterward with `agenix -d path/to/secret.age -i <identity>`

---

## Rekeying All Secrets

Only use this when every secret's recipients changed (e.g. a directory-wide key rotation), or when explicitly asked to rekey everything. Rekeying re-encrypts every `.age` file for the current set of recipients.

```bash
agenix -r -i <path/to/identity-key>
```

- The identity key passed with `-i` must already be a current recipient of the secrets being rekeyed — a newly generated key not yet listed in `secrets.nix` will cause a decryption error
- All `.age` files in the directory are re-encrypted, even ones whose recipients didn't change
- Files will always differ from before (age uses random nonces)
- Commit the rekeyed files to version control

---

## Key Facts

| Thing | Value |
|---|---|
| Working directory | Every command must run from the directory containing `secrets.nix` |
| Identity key | The agenix SSH identity key — pass with `-i <path/to/identity-key>` |
| Recipient keys | Declared per-secret in `secrets.nix` under `publicKeys` |
| Safe write method | `printf '...' \| agenix -e FILE -i KEY` (pipes work; agenix sets EDITOR to `cp /dev/stdin`) |
| Rekey one secret | `EDITOR=: agenix -e FILE -i KEY` (forces re-encryption without touching other files) |
| After writing | Verify with `agenix -d` immediately |

---

## Common Mistakes

**Forgetting to verify** → a failed encrypt leaves a zero-byte or corrupt file that will break service startup silently.

**Using plain `agenix -e FILE` to force a rekey** → if the plaintext is unchanged, agenix detects "no diff" and skips re-encryption. Use `EDITOR=: agenix -e FILE -i KEY` to force re-encryption of just that file.

**Running `agenix -r` when only one secret's recipients changed** → rekeys every `.age` file in the directory, creating diff noise for unrelated secrets. Use `EDITOR=: agenix -e FILE -i KEY` per affected file instead.

