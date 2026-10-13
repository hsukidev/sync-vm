# Remove the third-party SSH libraries

This spec moves an existing `sync-vm` implementation off three third-party modules, without changing what `sync-vm` does for the user. It is for the agent or engineer who owns the implementation that followed the earlier version of `docs/sync-architecture.md`.

## Why

Company policy allows only these dependencies:

- the Go standard library
- `golang.org/x/...` modules
- `github.com/fsnotify/fsnotify`

The earlier spec named three modules outside that list. Each one gets replaced by code in this repo, or by a `golang.org/x` package the repo already uses:

| Module | Used for | Replacement |
|---|---|---|
| `github.com/kevinburke/ssh_config` | Reading `~/.ssh/config` | New package `internal/sshconfig`. See "Parse the SSH config" in `docs/sync-architecture.md`. |
| `github.com/skeema/knownhosts` | `known_hosts` checks and the `HostKeyAlgorithms` lookup | `golang.org/x/crypto/ssh/knownhosts`, plus a lookup of about 40 lines. See "Check the host key". |
| `github.com/pkg/sftp` | Uploading the agent, and the SFTP server in `bootstrap` tests | New package `internal/sftp`. Tests run OpenSSH's own `sftp-server`. See "The SFTP client". |

Vendoring the modules with `go mod vendor` doesn't satisfy the policy, because vendored code is still third-party code. `fsnotify` stays.

`docs/sync-architecture.md` is the target design. This spec covers only the order of the work and how to prove nothing changed. Where the two disagree, the architecture doc wins for behavior, and this spec wins for sequencing.

## What must not change

The user sees the same `sync-vm`. In particular:

- The same Host alias resolves to the same `HostName`, `User`, `Port`, `IdentityFile` list, and `ProxyJump` as before, for every config the current tests cover.
- A known host connects without a prompt. An unknown host shows the same trust prompt. A changed key shows the same warning and doesn't retry.
- With `known_hosts` holding only an Ed25519 key, a server that also offers RSA still connects.
- Each passphrase is asked once per run.
- The agent installs to the same path, `.sync-vm/agent/<hash>/sync-vm-agent[.exe]`. A VM that already has the agent doesn't get it uploaded again.
- A VM without SFTP fails with the same message.

The migration adds a few intended changes, all from the architecture doc:

- `%h`, `%d`, `%u`, `%r`, and `%%` tokens in `HostName` and `IdentityFile` are expanded. An unknown token skips that value with a warning.
- A revoked host key (`@revoked` in `known_hosts`) fails that VM without retrying, with a message naming the key.
- `Include` stops with an error after 16 nested levels.

List any other difference you find in the PR description. Don't keep a difference only because the old library behaved that way. Don't add one silently, either.

## Before you start

1. Pull `main`. It has the updated `docs/sync-architecture.md`, the updated `issues/sync-architecture-tickets.md`, and this spec.
2. Run `go test ./...` and confirm it passes. Fix any existing failure before you continue, so a failure later points at the migration.
3. Run `go list -m all` and save the output. The final step compares against it.
4. Run `go mod why` for each of the three modules. If any other package imports one of them, add that package to the step that replaces the module.

## Steps

Each step ends with `go test ./...` passing and its own commit. Do them in order. The tests written in steps 2 to 4 run against the old library first, so they record today's behavior before any code changes.

### 1. Add the dependency test, with a list that only shrinks

Add a test that reads `go.mod` and fails when any `require` line names a module that is not under `golang.org/x/` and not `github.com/fsnotify/fsnotify`.

Give the test one more list, named for what it holds, such as `pendingRemoval`, containing the three modules. A module on that list passes. The test also fails when a module on that list no longer appears in `go.mod`, so each later step must remove its module from the list in the same commit.

Check the test by adding an unrelated module to `go.mod`, running the test, and seeing it fail. Then revert that change.

### 2. Replace `kevinburke/ssh_config` with `internal/sshconfig`

1. Write table tests for target resolution as it works now. Use config files in a temp home folder, and assert the resolved `HostName`, `User`, `Port`, `IdentityFile` list, and `ProxyJump`. Cover every case in the `internal/sshconfig` list in build step 5 of the architecture doc that the current code supports. Run them against the current code and confirm they pass.
2. Write `internal/sshconfig` per "Parse the SSH config", with its own table tests for the cases the old library didn't cover: tokens, the 16-level `Include` limit, and a bad `Port` naming the file and line.
3. Switch `bootstrap` to `sshconfig.Lookup`. The `-v` log of ignored settings comes from `Lookup`'s second return value.
4. Rerun the resolution tests from substep 1, unchanged. A failure there is a behavior change. Fix the code, or, if the architecture doc asks for the new behavior, update the expected value and list it in the PR description.
5. Remove the import, run `go mod tidy`, and delete `github.com/kevinburke/ssh_config` from `pendingRemoval`.

### 3. Replace `skeema/knownhosts`

1. Write tests for the host key check as it works now, with `known_hosts` fixtures for a plain entry, a hashed (`|1|...`) entry, a `[host]:2222` entry, a wildcard entry, an RSA entry, and a missing file. Assert the `HostKeyAlgorithms` value and the result of each check: known, unknown, or changed. Run them against the current code and confirm they pass.
2. Replace the lookup with the placeholder-key method in "Check the host key". Create the `knownhosts.New` callback once per `Dial`, and use the same callback for the lookup and for `ssh.ClientConfig.HostKeyCallback`.
3. Write new `known_hosts` lines with `knownhosts.Line` from `golang.org/x/crypto/ssh/knownhosts`, which is the function `skeema` called.
4. Add the `@revoked` case from the architecture doc.
5. Rerun the tests from substep 1, unchanged. The algorithm order for an RSA entry must be `rsa-sha2-512`, `rsa-sha2-256`, `ssh-rsa`.
6. Remove the import, run `go mod tidy`, and delete `github.com/skeema/knownhosts` from `pendingRemoval`.

### 4. Replace `pkg/sftp`

This step changes the test setup as well as the product code, so split it into two commits.

**Commit 1: the test server.** Change the in-process SSH server in the `bootstrap` tests to run OpenSSH's `sftp-server` as its `sftp` subsystem, in place of `pkg/sftp`'s server. Connect the subsystem channel to the process's stdin and stdout. Find the binary in the locations listed in build step 5 of the architecture doc. On Windows without the OpenSSH server feature, Git for Windows provides one. If none is found, skip the test with a message naming the paths it tried. The product code still uses `pkg/sftp`'s client here. The install tests must pass against the real server before the client changes.

**Commit 2: the client.**

1. Write `internal/sftp` per "The SFTP client", with the tests listed for it in the architecture doc.
2. Switch the install step in `bootstrap` to `internal/sftp`. Keep the steps the same: `Stat`, `MkdirAll`, write `.tmp`, `Chmod` to `0755` on Linux and macOS, then `PosixRename` when offered, or `Remove` and `Rename` otherwise.
3. Map a refused `sftp` subsystem to the same "doesn't offer SFTP" error as before.
4. Rerun the install tests: a first dial uploads the agent, and a second dial finds it and uploads nothing.
5. Remove the import, run `go mod tidy`, and delete `github.com/pkg/sftp` from `pendingRemoval`.

### 5. Close out

1. `pendingRemoval` is now empty. Delete it from the dependency test, leaving only the allow rule.
2. Run `go list -m all` and compare it with the list saved before you started. Confirm the three modules are gone, along with any modules that only they pulled in, such as `github.com/kr/fs`. Confirm nothing new appeared.
3. Run `go test -race ./...`.
4. Run `scripts/build-dist.sh` and confirm every agent build still compiles.
5. Repeat the real-VM checks from build step 5 of the architecture doc:
   - Sync to a real Linux VM and a real Windows VM from one `sync-vm` process, with one passphrase-protected key. Confirm the passphrase is asked once.
   - Delete `.sync-vm/agent/` on one VM, so the next run installs the agent over the new SFTP client.
   - On the Windows VM, test with cmd as the login shell, then with PowerShell.

## Done when

- `go.mod` requires only `golang.org/x/...` modules and `github.com/fsnotify/fsnotify`, and the dependency test enforces it.
- Every test written in steps 2 to 4 still passes, unchanged except where the PR description lists an intended change.
- The real-VM checks in step 5 pass.
