# Apono Fork of AWS Session Manager Plugin

Fork of [aws/session-manager-plugin](https://github.com/aws/session-manager-plugin) that makes the plugin importable as a Go library.

## Why

The upstream repo is a GOPATH project with no `go.mod`. It can't be imported via Go modules. This fork adds module support so the Apono connector can run the SSM port-forwarding protocol in-process — no external binary, no Alpine/glibc compatibility issues.

## Changes from upstream

| Change | Files | Why |
|--------|-------|-----|
| Add `go.mod` / `go.sum` | 2 new files | Enable Go module imports |
| Remove `vendor/` | Directory deleted | Old GOPATH vendor conflicts with Go modules |
| `uuid.CleanHyphen` → `uuid.FormatCanonical` | 5 files | Constant renamed in module-published version of `twinj/uuid` |
| `os.Exit(0)` → `return` in `Stop()` methods | 7 files | `os.Exit` kills the host process when used as a library |

## Branches

- **`mainline`** — untouched upstream (synced from aws/session-manager-plugin)
- **`apono-main`** — our patched version (tags go here)

## Tagging scheme

Tags map directly to upstream versions with the 4th number dropped:

- Upstream `1.2.804.0` → our tag `v1.2.804`
- Upstream `1.2.810.0` → our tag `v1.2.810`

## Updating to a new upstream release

```bash
# 1. Fetch upstream
git fetch origin mainline --tags

# 2. Rebase our patches onto the new tag
git checkout apono-main
git rebase <new-upstream-tag>

# 3. Check for new os.Exit or stdout writes in new files
grep -rn 'os\.Exit' src/ --include='*.go' | grep -v _test.go | grep -v versiongenerator
grep -rn 'fmt.*os\.Stdout' src/ --include='*.go' | grep -v _test.go

# 4. Fix any new occurrences (os.Exit → return, os.Stdout → os.Stderr)

# 5. Tag and push
git tag -a v1.2.XXX -m "Based on upstream aws/session-manager-plugin 1.2.XXX.0"
git push --force origin apono-main
git push origin v1.2.XXX

# 6. Update connector go.mod
# replace github.com/aws/session-manager-plugin => github.com/apono-io/session-manager-plugin v1.2.XXX
```

## Usage in the connector

The connector imports via a `replace` directive in `go.mod`:

```
replace github.com/aws/session-manager-plugin => github.com/apono-io/session-manager-plugin v1.2.804
```

This keeps the original import paths (`github.com/aws/session-manager-plugin/src/...`) unchanged — all internal cross-references within the library work without modification.

## Known side effects when running as a library

- **stdout writes** — The library writes status messages to `os.Stdout`. The connector redirects `os.Stdout = os.Stderr` before calling it, because hashicorp/go-plugin uses stdout as gRPC transport.
- **signal handlers** — Registers `signal.Notify` for SIGINT/SIGQUIT. Harmless since the connector plugin runs in its own process.
- **seelog logger** — Creates log files at `/usr/local/sessionmanagerplugin/logs/`. Fails silently in containers.
