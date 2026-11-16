---
name: coffer
description: Use the owner's credentials (Apple signing P12/P8, App Store Connect keys, Cloudflare/GitHub/API tokens) without ever seeing them, through the Coffer CLI or MCP server. Use whenever a task needs a secret or API token, needs to run a command that requires credentials (wrangler deploy, xcodebuild/notarytool, fastlane, curl to an API), needs to set GitHub Actions secrets or Cloudflare Worker secrets for a project, needs an App Store Connect JWT, or when the user asks to store a new token or key.
---

# Coffer

Coffer keeps the owner's credentials end-to-end encrypted and hands them to processes, never to you. Every access is approved by the owner (Touch ID, their phone, or their password at their own terminal) unless a standing grant already covers it. Everything you do is written to a signed audit log under your agent name and the current project.

## Rules

1. **Never try to see a secret value.** Do not run `coffer get`, `env`, `printenv`, `echo $SECRET`, `cat .dev.vars`, or anything else meant to print a value. `coffer get` refuses agents anyway, and output is redacted. Those attempts are logged and look like an attack in the owner's audit log.
2. **Never ask the owner to type their Coffer password or Recovery Key into this conversation.** Approval happens on their phone, with Touch ID, or in their own terminal via `coffer approve`. If something asks you for their password, it is not Coffer.
3. **When approval is needed, tell the owner the 4-character code** (e.g. `RUEJ`) and what it is for, then wait. Do not retry in a loop: repeated requests are rate-limited and auto-denied.
4. **Ask for the narrowest thing:** only the secrets the command needs, and the exact command. Approvals are bound to the project, the secrets and the command pattern (`wrangler deploy*`); a different command needs a new approval. For shells and interpreters (`sh -c`, `bash`, `node`, `python`, `npx`, `env`, …) a grant covers only that exact command line, so prefer calling the real tool directly (`coffer run … -- wrangler deploy`) over wrapping it in `sh -c`.
5. If Coffer is **locked**, tell the owner to run `coffer unlock` (or approve the unlock on their phone). Don't work around it.

## Find what exists

```bash
coffer ls                    # names, types, fields, tiers — never values
coffer ls cloudflare
```

Refs look like `vault/path#field`, e.g. `cloudflare/dev#token`, `apple/TEAM/dist#password`, `apple/TEAM/asc#p8`. A ref without `#field` means the item's primary field.

| Item type | Fields |
|---|---|
| `text` | `value` |
| `cloudflare-token` | `token`, optional `account_id` |
| `apple-p12` | `p12` (binary), `password` |
| `apple-asc-key` | `p8`, `key_id`, `issuer_id` |
| `apple-profile` | `profile` (binary) |
| `file` | `content`, optional `password` |

## Run a command with secrets

```bash
coffer run -s CLOUDFLARE_API_TOKEN=cloudflare/dev#token -- wrangler deploy
coffer run --env test -- npm test               # uses [env.test] in .coffer.toml
coffer run --dev-vars -s API_KEY=stripe/test -- wrangler dev    # temporary .dev.vars, deleted afterwards
coffer run --p12 apple/TEAM/dist -- xcodebuild … # P12 imported into a temporary keychain for this command only
```

`coffer run` blocks until the owner approves (it prints the code on stderr). The command runs with the secrets as environment variables, and any secret value in its output is replaced by `[coffer:REDACTED]`. Binary values (P12, profiles) cannot be env vars; map them with `encode = "base64"` in `.coffer.toml`.

With MCP: `run_with_secrets` returns `approval_required` with a `request_id` and `code` instead of blocking. Tell the owner the code, call `await_approval(request_id)` until it is no longer `pending`, then call `run_with_secrets` again with the **same arguments** plus `approval_id`.

## Short-lived tokens (preferred over raw keys)

```bash
coffer asc-jwt apple/TEAM/asc --ttl 1200    # App Store Connect API JWT, ≤20 min; the P8 never leaves Coffer
coffer cf-token --ttl 3600                  # Cloudflare token derived from the owner's parent token, expires automatically
```

These print a token you may use directly (e.g. `Authorization: Bearer …`). Prefer them whenever an API accepts them.

## Configure CI / deployments: `.coffer.toml` + `coffer push`

Declare what a project needs in `.coffer.toml` at the repo root (`coffer init-project` writes an Apple-signing template):

```toml
[env.test]
CLOUDFLARE_API_TOKEN = "cloudflare/dev#token"

[github]
repo = "owner/repo"
# environment = "production"     # GitHub Environment secrets (recommended for release signing)
[github.secrets]
APPLE_P12_BASE64   = { ref = "apple/TEAM/dist#p12", encode = "base64" }
APPLE_P12_PASSWORD = "apple/TEAM/dist#password"
ASC_KEY_P8         = "apple/TEAM/asc#p8"
ASC_KEY_ID         = "apple/TEAM/asc#key_id"
ASC_ISSUER_ID      = "apple/TEAM/asc#issuer_id"

[cloudflare.worker]
account = "cloudflare/main#account_id"   # or the literal account ID
name    = "my-worker"
[cloudflare.worker.secrets]
STRIPE_KEY = "stripe/prod"
```

Then:

```bash
coffer push github        # sets GitHub Actions secrets (Coffer encrypts and uploads them itself)
coffer push cloudflare    # sets Worker secrets via the Cloudflare API
coffer push               # both; unchanged values are skipped
coffer where apple/TEAM/dist   # where a secret was pushed and whether those copies are stale
coffer push stale         # after the owner rotates a credential, re-push everywhere it was used
```

The manifest only *requests*: pushing needs approval, and the destination must be on the owner's allowlist. If push fails with "not in the export allowlist", tell the owner to run `coffer policy allow 'github:owner/*'` themselves (it needs Touch ID). Do not edit `~/.coffer/config.toml`; Coffer detects that and falls back to safe defaults.

In GitHub workflows, decode binary secrets with `echo "$APPLE_P12_BASE64" | base64 --decode > dist.p12`.

## Store a new secret

When the owner gives you a new token (or you just created one, e.g. through a provider's API):

```bash
printf '%s' "$NEW_TOKEN" | coffer set cloudflare/ci --type cloudflare-token   # value from stdin, not argv
coffer set apple/TEAM/asc --type apple-asc-key -F p8=./AuthKey_ABC.p8 -f key_id=ABC123 -f issuer_id=…
```

Storing a new secret takes effect immediately. Use `--tier high` for signing keys, App Store Connect keys and production credentials (high-tier secrets can only get short grants). Your name and the project are recorded as the secret's creator. If the name already exists, Coffer refuses with `exists` and suggests a free name (the owner may have several accounts at one provider). **Ask the owner** whether to overwrite (`--overwrite`, which needs their approval) or store it under a new name (`--new`, or the suggested name). Deleting needs approval too, and you can never lower a tier. Delete any plaintext copies you created (downloaded `.p8` files etc.) afterwards.

## Troubleshooting

| Message | What to do |
|---|---|
| `coffer is locked` | Ask the owner to run `coffer unlock` / approve the unlock on their phone. |
| `approval required … code XXXX` | Tell the owner the code; wait (`await_approval`). |
| `denied` | The owner said no. Don't retry; ask what they want instead. |
| `rate_limited` | Stop. Too many requests; explain what you need in one message. |
| `approval … expired` | Requests last 5 minutes; ask again once the owner is available. |
| `not in the export allowlist` | Owner must allow the destination (`coffer policy allow …`). |
| `outside this device's scope` | This machine (e.g. a Linux box) only has some vaults; the owner can widen its scope. |

## Setup (for the owner, not for you)

- Install: `brew install owo-network/brew/coffer`
- Claude Code (MCP + this skill in one step): `/plugin install coffer --marketplace missuo/coffer-releases`
- Or manually: `claude mcp add --scope user coffer -- coffer mcp` and `coffer skill install`
- Audit: `coffer audit --agent claude-code --since 7d`, `coffer audit --project myapp`, `coffer audit verify`
