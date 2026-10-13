# Dependency migration: Tickets

Tickets for `docs/dependency-migration.md`, which moves an existing `sync-vm` implementation off `github.com/kevinburke/ssh_config`, `github.com/skeema/knownhosts`, and `github.com/pkg/sftp`. Company policy allows only the Go standard library, `golang.org/x/...`, and `github.com/fsnotify/fsnotify`.

Read the migration spec before starting any ticket. It owns the order of work and the list of behavior that must not change. `docs/sync-architecture.md` owns the target design: "Parse the SSH config", "Check the host key", and "The SFTP client".

Every ticket ends with `go test ./...` passing. A ticket that removes a module also removes it from the dependency test's pending list in the same commit.

Frontier at the start: 01 and 04. After 01: 02 and 03 in parallel, and 05 once 04 is also done. Tickets 02, 03, and 05 all edit `go.mod` and the pending list, so expect small merge conflicts there.

| # | Title | Blocked by |
|---|---|---|
| 01 | Dependency test with a list that only shrinks | None |
| 02 | Replace `kevinburke/ssh_config` with `internal/sshconfig` | 01 |
| 03 | Replace `skeema/knownhosts` | 01 |
| 04 | Move the `bootstrap` tests onto OpenSSH's `sftp-server` | None |
| 05 | Replace the `pkg/sftp` client with `internal/sftp` | 01, 04 |
| 06 | Close out and check on real VMs | 02, 03, 05 |

---

## 01: Dependency test with a list that only shrinks

**What to build:** A test that makes the dependency policy part of the build. It fails when `go.mod` requires a module outside the allowed set. The three modules being removed sit on a pending list, so the build stays green until each one is replaced, and the list can only get shorter.

**Blocked by:** None (can start immediately).

**Status:** ready-for-agent

- [ ] Baseline recorded before any change: `go test ./...` passes, and the output of `go list -m all` is saved for ticket 06.
- [ ] `go mod why` run for each of the three modules. Any package other than the ones the migration spec names is added to the matching ticket.
- [ ] The test reads `go.mod` and fails for any `require` that isn't under `golang.org/x/` and isn't `github.com/fsnotify/fsnotify`, unless the module is on the pending list.
- [ ] The pending list holds exactly the three modules, under a name that says what it holds.
- [ ] The test also fails when a module on the pending list no longer appears in `go.mod`.
- [ ] Verified by adding an unrelated module, seeing the test fail, then reverting.

---

## 02: Replace `kevinburke/ssh_config` with `internal/sshconfig`

**What to build:** `sync-vm` resolves every Host alias to the same `HostName`, `User`, `Port`, `IdentityFile` list, and `ProxyJump` as before, using the in-house parser. The only intended changes are the ones the migration spec lists: `%` tokens are expanded, and `Include` stops after 16 nested levels.

**Blocked by:** 01

**Status:** ready-for-agent

- [ ] Resolution tests written first and passing against the old library. They use config files in a temp home folder and cover every supported case from build step 5 of the architecture doc that the current code handles.
- [ ] `internal/sshconfig` built per "Parse the SSH config", with table tests for tokens, an unknown token, the 16-level `Include` limit, and a bad `Port` naming the file and line.
- [ ] `bootstrap` resolves targets through `sshconfig.Lookup`. The `-v` log of ignored settings comes from `Lookup`.
- [ ] The resolution tests from the first item pass unchanged. Any expected value that had to change is an intended change from the architecture doc, listed in the PR description.
- [ ] The import is gone, `go mod tidy` has run, and the module is off the pending list.

---

## 03: Replace `skeema/knownhosts`

**What to build:** Host key checks behave as before, using only `golang.org/x/crypto/ssh/knownhosts`. A known host connects without a prompt, an unknown host gets the same trust prompt, and a changed key shows the same warning without retrying. A VM whose `known_hosts` entry is Ed25519 still connects when it also offers RSA. The intended new behavior is that a revoked key fails without retrying, with a message naming the key.

**Blocked by:** 01

**Status:** ready-for-agent

- [ ] Tests written first and passing against the old library, with `known_hosts` fixtures for a plain entry, a hashed entry, a `[host]:2222` entry, a wildcard entry, an RSA entry, and a missing file. They assert `HostKeyAlgorithms` and each check result: known, unknown, or changed.
- [ ] The algorithm lookup uses the placeholder-key method from "Check the host key". One `knownhosts.New` callback per `Dial` serves both the lookup and `HostKeyCallback`.
- [ ] An RSA entry gives `rsa-sha2-512`, `rsa-sha2-256`, `ssh-rsa`, in that order. An unknown host leaves `HostKeyAlgorithms` unset.
- [ ] New lines are written with `knownhosts.Line`. The trust prompt and the file and folder creation are unchanged.
- [ ] An `@revoked` key fails that VM without retrying and names the key, with a test.
- [ ] The tests from the first item pass unchanged.
- [ ] The import is gone, `go mod tidy` has run, and the module is off the pending list.

---

## 04: Move the `bootstrap` tests onto OpenSSH's `sftp-server`

**What to build:** The install tests in `bootstrap` run against OpenSSH's real `sftp-server` in place of `pkg/sftp`'s test server, while the product code still uses the old client. This proves the tests check real SFTP behavior before the client changes in ticket 05.

**Blocked by:** None (can start immediately).

**Status:** ready-for-agent

- [ ] The in-process SSH server's `sftp` subsystem starts `sftp-server` and connects the channel to the process's stdin and stdout.
- [ ] The binary is found in the locations listed in build step 5 of the architecture doc, including the Git for Windows copy.
- [ ] When none is found, the tests skip with a message naming every path tried.
- [ ] Existing install tests pass against the real server: a first dial uploads the agent, a second dial uploads nothing, and a refused subsystem gives the "doesn't offer SFTP" error.
- [ ] Nothing in the tests imports `pkg/sftp`'s server anymore.

---

## 05: Replace the `pkg/sftp` client with `internal/sftp`

**What to build:** The agent installs through the in-house SFTP version 3 client, to the same path, with the same steps and the same errors as before. A VM that already has the agent doesn't get it uploaded again.

**Blocked by:** 01, 04

**Status:** ready-for-agent

- [ ] `internal/sftp` built per "The SFTP client": `NewClient`, `Stat`, `MkdirAll`, `WriteFile` in 32 KB writes, `Chmod`, `Remove`, `Rename`, `PosixRename`, with one request in flight at a time.
- [ ] Client tests against the real `sftp-server`: `Stat` of a missing path matching `fs.ErrNotExist`, nested `MkdirAll`, a 1 MB `WriteFile` that reads back identical, `Chmod`, and both renames.
- [ ] A fake-server test for a bad status, a wrong request ID, and a refused subsystem.
- [ ] The install step runs `Stat`, `MkdirAll`, write `.tmp`, `Chmod` to `0755` on Linux and macOS, then `PosixRename` when offered, or `Remove` and `Rename` otherwise.
- [ ] A refused subsystem maps to the same "doesn't offer SFTP" error.
- [ ] The install tests from ticket 04 pass unchanged.
- [ ] The import is gone, `go mod tidy` has run, and the module is off the pending list.

---

## 06: Close out and check on real VMs

**What to build:** The migration is finished and proven on real VMs. `go.mod` holds only allowed modules, the dependency test has no exceptions left, and a user syncing to Linux and Windows VMs sees no difference.

**Blocked by:** 02, 03, 05

**Status:** ready-for-agent

- [ ] The pending list is deleted from the dependency test, leaving only the allow rule.
- [ ] `go list -m all` compared with the baseline from ticket 01: the three modules and anything only they pulled in are gone, and nothing new appeared.
- [ ] `go test -race ./...` passes.
- [ ] `scripts/build-dist.sh` builds every agent.
- [ ] Real-VM check: one `sync-vm` process syncs to a Linux VM and a Windows VM with one passphrase-protected key, and asks for the passphrase once.
- [ ] Real-VM check: with `.sync-vm/agent/` deleted on each VM, the next run installs the agent through the new client.
- [ ] Real-VM check: the Windows VM works with cmd as the login shell, then with PowerShell.
- [ ] The PR description lists every behavior change, matching the migration spec's intended changes.
