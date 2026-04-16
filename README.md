
# Session Manager Plugin (Apono Fork)

> **This is an Apono fork** of [aws/session-manager-plugin](https://github.com/aws/session-manager-plugin) that adds Go module support for library import. See [Apono Fork Details](#apono-fork-details) below.

This plugin helps you to use the AWS Command Line Interface (AWS CLI) to start and end sessions to your managed instances. Session Manager is a capability of AWS Systems Manager.

## Overview

Session Manager is a fully managed AWS Systems Manager capability that lets you manage your Amazon Elastic Compute Cloud (Amazon EC2) instances, on-premises instances and virtual machines. Session Manager provides secure and auditable instance management without the need to open inbound ports. When you use the Session Manager plugin with the AWS CLI to start a session, the plugin builds the websocket connection to your managed instances.

### Prerequisites

Before using Session Manager, make sure your environment meets the following requirements. [Complete Session Manager prerequisites](http://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-prerequisites.html).

### Starting a session

For information about starting a session using the AWS CLI, see [Starting a session (AWS CLI)](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-sessions-start.html#sessions-start-cli).

### Troubleshooting

For information about troubleshooting, see [Troubleshooting Session Manager](http://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-troubleshooting.html).

### Version Compatibility

The default compiled version is 1.3.0.0, which enables all the latest features and functionality for local builds. Official releases use the 1.2.x.x versioning scheme. If you are building locally, we recommend keeping the default version (1.3.0.0) in both `VERSION` and `src/version/version.go` to ensure access to all available features.

### Working with Docker

To build the Session Manager plugin in a `Docker` container, complete the following steps:

1. Install [`docker`](https://docs.docker.com/engine/install/centos/)

2. Build the `docker` image
```
docker build -t session-manager-plugin-image .
```
3. Build the plugin
```
docker run -it --rm --name session-manager-plugin -v `pwd`:/session-manager-plugin session-manager-plugin-image make release
```

### Working with Linux

To build the binaries required to install the Session Manager plugin, complete the following steps.

1. Install `golang`

2. Install `rpm-build` and `rpmdevtools`

3. Install `gcc 8.3+` and `glibc 2.27+`

4. Run `make release` to build the plugin for Linux, Debian, macOS and Windows.

5. Change to the directory of your local machine's operating system architecture and open the `session-manager-plugin` directory. Then follow the installation procedure that applies to your local machine. For more information, see [Install the Session Manager plugin for the AWS CLI](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html). If the machine you're building the plugin on differs from the machine you plan to install the plugin on you will need to copy the `session-manager-plugin` binary to the appropriate directory for that operating system.

```
Linux - /usr/local/sessionmanagerplugin/bin/session-manager-plugin

macOS - /usr/local/sessionmanagerplugin/bin/session-manager-plugin

Windows - C:\Program Files\Amazon\SessionManagerPlugin\bin\session-manager-plugin.exe
```

The `ssmcli` binary is available for some operating systems for testing purposes only. The following is an example command using this binary.

```
./ssmcli start-session --instance-id i-1234567890abcdef0 --region us-east-2
```

### Directory structure

Source code

* `sessionmanagerplugin/session` contains the source code for core functionalities
* `communicator/` contains the source code for websocket related operations
* `vendor/src` contains the vendor package source code
* `packaging/` contains rpm and dpkg artifacts
* `Tools/src` contains build scripts

## Feedback

Thank you for helping us to improve the Session Manager plugin. Please send your questions or comments to the [Systems Manager Forum](https://forums.aws.amazon.com/forum.jspa?forumID=185&start=0)

## Apono Fork Details

### Why

The upstream repo is a GOPATH project with no `go.mod`. It can't be imported via Go modules. This fork adds module support so the Apono connector can run the SSM port-forwarding protocol in-process — no external binary, no Alpine/glibc compatibility issues.

### Changes from upstream

| Change | Files | Why |
|--------|-------|-----|
| Add `go.mod` / `go.sum` | 2 new files | Enable Go module imports |
| Remove `vendor/` | Directory deleted | Old GOPATH vendor conflicts with Go modules |
| `uuid.CleanHyphen` → `uuid.FormatCanonical` | 5 files | Constant renamed in module-published version of `twinj/uuid` |
| `os.Exit(0)` → `return` in `Stop()` methods | 7 files | `os.Exit` kills the host process when used as a library |

### Branches

- **`mainline`** — untouched upstream (synced from aws/session-manager-plugin)
- **`apono-main`** — our patched version (tags go here)

### Tagging scheme

Tags map directly to upstream versions with the 4th number dropped:

- Upstream `1.2.804.0` → our tag `v1.2.804`
- Upstream `1.2.810.0` → our tag `v1.2.810`

### Updating to a new upstream release

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

### Known side effects when running as a library

- **stdout writes** — The library writes status messages to `os.Stdout`. The connector redirects `os.Stdout = os.Stderr` before calling it, because hashicorp/go-plugin uses stdout as gRPC transport.
- **signal handlers** — Registers `signal.Notify` for SIGINT/SIGQUIT. Harmless since the connector plugin runs in its own process.
- **seelog logger** — Creates log files at `/usr/local/sessionmanagerplugin/logs/`. Fails silently in containers.

## License

The session-manager-plugin is licensed under the Apache 2.0 License.
