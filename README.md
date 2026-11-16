# coffer-releases

Signed and notarized release binaries of **Coffer**: end-to-end encrypted credentials for you and your AI agents.

| Asset | Platform |
|---|---|
| `coffer-<version>-macos-universal.tar.gz` | macOS 13+ (Apple Silicon + Intel). Signed with a Developer ID, hardened runtime, notarized by Apple |
| `coffer-<version>-linux-x86_64.tar.gz` | Linux x86_64, static (musl) |
| `coffer-<version>-linux-aarch64.tar.gz` | Linux arm64, static (musl) |
| `SHA256SUMS` | checksums of the archives |

Verify:

```bash
shasum -a 256 -c SHA256SUMS --ignore-missing
codesign --verify --strict --verbose=2 coffer          # macOS
codesign -dv coffer 2>&1 | grep -E "Authority|TeamIdentifier|Runtime"
```

## Install

```bash
brew install owo-network/brew/coffer          # macOS (signed + notarized) and Linux
```

## Use with AI agents

The MCP server and the agent skill ship **inside the `coffer` binary**, and as a Claude Code plugin in this repo.

**Claude Code plugin** (MCP server + skill in one step; needs `coffer` on your PATH):

```
/plugin install coffer --marketplace missuo/coffer-releases
```

or: `/plugin marketplace add missuo/coffer-releases` then `/plugin install coffer@coffer`.

**Manually** (any agent):

```bash
coffer skill install                         # ~/.claude/skills/coffer/SKILL.md (or --target codex / <dir>)
claude mcp add --scope user coffer -- coffer mcp
```

The skill is also in [`plugin/skills/coffer/SKILL.md`](plugin/skills/coffer/SKILL.md) and attached to every release as `coffer-<version>-SKILL.md`.

Source code is not published here.
