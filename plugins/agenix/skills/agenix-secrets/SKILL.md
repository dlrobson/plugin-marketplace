---
name: agenix-secrets
description: This skill should be used when the user mentions "agenix", asks about ".age files", "secrets.nix", or age-encrypted NixOS secrets, or asks to "encrypt an agenix secret", "update an agenix secret", "add a new agenix secret", "rekey agenix secrets", "decrypt a .age file", or any task involving reading, writing, or rekeying age-encrypted agenix secrets.
version: 1.0.0
---

# agenix Secret Management

agenix encrypts NixOS secrets using `age`. Each `.age` file is encrypted for one or more SSH public key recipients declared in `secrets.nix`. The agenix CLI handles decryption; `age` itself handles encryption.

**Critical rule:** Never use `agenix -e` for scripted or automated writes — it silently produces an empty file when `$EDITOR` is not a real interactive editor. Always write secrets using `age` directly.

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

**Three steps — always do all three:**

### Step 1: Look up recipient keys

Open `secrets.nix` (typically at the root of the secrets directory) and find the entry for this secret. Collect all `publicKeys` values listed for the target secret.

```nix
# Example secrets.nix entry:
"vms/my-secret.age".publicKeys = [
  "ssh-ed25519 AAAAC3Nza..."   # host key
  "ssh-ed25519 AAAAC3Nzb..."   # user key
];
```

### Step 2: Encrypt with age via nix-shell

`age` is not in PATH on NixOS by default. On classic (non-flake) systems use `nix-shell`; on flake-based systems use `nix shell`:

```bash
# non-flake
nix-shell -p age --run "
  printf 'YOUR_SECRET_VALUE\n' | age \
    -r 'ssh-ed25519 AAAAC3Nza...' \
    -r 'ssh-ed25519 AAAAC3Nzb...' \
    -o path/to/secret.age
"

# flake-based
nix shell nixpkgs#age --command sh -c "
  printf 'YOUR_SECRET_VALUE\n' | age \
    -r 'ssh-ed25519 AAAAC3Nza...' \
    -r 'ssh-ed25519 AAAAC3Nzb...' \
    -o path/to/secret.age
"
```

- Use `printf` (not `echo`) to control trailing newlines precisely
- Pass one `-r` flag per recipient key from `secrets.nix`
- The `-o` path overwrites the existing file if it exists

### Step 3: Verify immediately

Always decrypt right after writing to confirm the file is not empty and the value is correct:

```bash
agenix -d path/to/secret.age -i <path/to/identity-key>
```

If this outputs nothing or errors, the write failed — repeat from Step 1.

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
| `age` in PATH? | No — use `nix-shell -p age --run "..."` |
| Safe write method | `age` directly (never `agenix -e` for scripted use) |
| After writing | Verify with `agenix -d` immediately |

---

## Common Mistakes

**`agenix -e` with a scripted EDITOR** → silently writes an empty `.age` file. Only use `agenix -e` when sitting at an interactive terminal with a real editor.

**Forgetting to verify** → a failed encrypt leaves a zero-byte or corrupt file that will break service startup silently.

**Running `agenix -r` from the wrong directory** → rekey must be run from the directory containing `secrets.nix`, not a subdirectory or parent.

