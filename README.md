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

Source code is not published here.
