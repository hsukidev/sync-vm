# sync-vm Sync Architecture: Tickets

Tickets for implementing `docs/sync-architecture.md`. Each ticket is a vertical slice that can be verified on its own. Read the architecture doc before starting any ticket; it is the source of truth for every message, rule, and output string named here.

Until ticket 13, the host engine talks to the agent in-process over an in-memory pipe, behind a dialer interface that ticket 01 introduces. Ticket 13 puts the real SSH dialer behind the same interface.

Frontier at the start: 01. After 01: 02, 05, 07, 08, 09, 11 can run in parallel.

| # | Title | Blocked by |
|---|---|---|
| 01 | Tracer: one-shot sync of a git repo into a folder | None |
| 02 | CLI and repo config | 01 |
| 03 | `init` and destination safety checks | 02 |
| 04 | VM-edit exception, overwritten copies, and `-reset` | 01, 02 |
| 05 | Watch mode | 01 |
| 06 | Reconnect and safety guards | 05 |
| 07 | Agent durability and throughput | 01 |
| 08 | Host hash cache and fast startup | 01 |
| 09 | Copy instead of resending | 01 |
| 10 | Allowlist and listing filters | 02 |
| 11 | Executable bit and CRLF warning | 01 |
| 12 | Several VMs in one process | 05, 06 |
| 13 | SSH bootstrap to a real VM | 03 |
| 14 | SSH login prompts | 13 |
| 15 | SSH connection robustness | 13 |

---

## 01: Tracer: one-shot sync of a git repo into a folder

**What to build:** The thinnest complete path from a host git repo to a VM folder. The host lists the repo's files through git, opens a session with an agent, receives the agent's manifest, runs `Diff`, sends puts and deletes, and the agent applies them and records what it wrote in `state.json`. After one session, the VM folder holds exactly the repo's files. Sets up the Go module, the two binaries, and the package layout from "Modules", plus a dialer interface the engine uses to reach an agent, with an in-process implementation for tests.

**Blocked by:** None (can start immediately).

**Status:** ready-for-agent

- [ ] Go module with `cmd/sync-vm`, `cmd/sync-vm-agent`, and the six internal packages from "Modules" (empty where not yet needed).
- [ ] `protocol`: frame format (4-byte length, `M`/`D` tag, 64 MB cap), JSON messages `Hello`, `Manifest`, `Put` + 1 MB data frames, `Delete`, `Ack`, `Fatal`; `Conn` with concurrent `Send`/`Recv`; a received `Put` exposes content as an `io.Reader`.
- [ ] `RelPath` parser rejecting every case listed under Data, including `.sync-vm` as a first segment in any case; paths parsed on decode on both sides.
- [ ] `Manifest` split into messages of at most 10,000 files with `more`; a 25,000-file manifest round-trips as three messages.
- [ ] `engine.Diff` implements all eight rows, with `kept` output and sorted results (deletes first); `AllPaths` helper. One unit test per row, with and without `reset`.
- [ ] Basic `Tree.List` from `git ls-files -s` and `git ls-files -o --exclude-standard` (with `-z`), hashing each file with SHA-256; returns an error, never an empty map, when git fails.
- [ ] Agent: receives `Hello`, opens the root with `os.OpenRoot`, loads `state.json`, sends `Manifest`, applies `Put` (write, verify hash, rename) and `Delete` (remove file and empty parents), records `size/modTime/hash/synced/hostSize/hostModTime`, sends `Ack`, saves `state.json`. Exits when stdin closes.
- [ ] Engine runs one session: dial via the dialer interface, `Hello`, collect manifest, `List`, `Diff`, send, apply `Ack`s to the in-memory picture of the VM.
- [ ] Round-trip test of every message, a put larger than one data frame, and traversal / `.sync-vm/...` paths rejected on decode.
- [ ] End-to-end in-process test: create, edit, and delete files in a temp repo; run a session; the VM folder matches.

---

## 02: CLI and repo config

**What to build:** The `sync-vm` command line and the per-repo config. A user inside a git repo can see help and version, list and remove saved VMs, and run `sync-vm [<target>...]` to sync the saved targets (using the in-process dialer until ticket 13). Outside a git repo, every command but help and version fails with the documented message.

**Blocked by:** 01

**Status:** ready-for-agent

- [ ] Argument parsing follows "Commands" exactly: flags anywhere, `--` ends flags, single- and double-dash forms, `-h` beats everything, then `-version`; `init`, `vm list|remove`, `allowlist list|add|remove`, otherwise sync with targets; per-command `flag.FlagSet` with `ContinueOnError`; errors print message plus that command's usage and exit 2; word counts checked per command.
- [ ] Usage text for `sync-vm` matches the doc; each command has its own `-h` usage.
- [ ] `-version` prints `sync-vm <version> (protocol <n>)`, `version` set by `-ldflags`, `dev` otherwise.
- [ ] `repo.Find` uses `git rev-parse --show-toplevel` and `--absolute-git-dir`; a worktree returns its own git folder. Failure prints `sync-vm must run inside a git repository.`
- [ ] `repo.ParseTarget` splits at the first `:` and defaults the path to `~/<repo folder name>`; tested with a Linux path, a Windows path, and no path.
- [ ] `Load`/`Save` of `<git-dir>/sync-vm.json` (temp file + rename); missing file is an empty config.
- [ ] `vm list` output and empty message as documented; `vm remove` message, and error listing saved targets for an unknown one.
- [ ] Sync command: no VMs → documented message; unknown target → error listing saved targets and suggesting `init`.
- [ ] Table test of parsing: flags before/after/between words, `--` before a dash word, `--force` = `-force`, `-reset` with `init` rejected with status 2, wrong word counts, missing/unknown subcommands, `-h` and `-version` exit 0 outside a repo.

---

## 03: `init` and destination safety checks

**What to build:** `sync-vm init <target>[:<path>] [-force]` sets up a VM folder and saves the target. The agent refuses dangerous or wrong destinations before touching anything, and a normal sync only accepts a folder this repo set up. Two `sync-vm` processes can't sync the same folder at once.

**Blocked by:** 02

**Status:** ready-for-agent

- [ ] `init` flow per the doc: refuses target names `init`, `vm`, `allowlist`; refuses a saved target with a different folder; creates the repo ID (32 hex chars from `crypto/rand`) on first use; sends `Hello` with `init`/`force`; saves `Manifest.root` on success with the documented message; saves nothing on `Fatal`. Rerunning with the same folder changes nothing.
- [ ] Agent destination checks 1–5 in order: `~` expansion (only `~`, `~/`, `~\`), absolute path, symlink resolution (nearest existing parent), system-folder list per OS (built from injected environment values so Windows rows test on any OS), and the check 5 table for sync, `init`, and `init -force`.
- [ ] Rejections A–D with the documented messages; the host fills in `<target>`.
- [ ] A normal run never creates a folder or changes an owner; a `-force` takeover keeps every record; a damaged `state.json` gives D, and `init -force` replaces it.
- [ ] Exclusive OS lock on `<root>/.sync-vm/lock` (`flock` / `LockFileEx`); a held lock gives `Fatal`.
- [ ] Engine treats `Fatal` as final: prints the reason, returns an error, no reconnect.
- [ ] Tests: each check with every command, a file instead of a folder, every cell of the check 5 table, `init` twice with different paths refused, reserved target names refused.

---

## 04: VM-edit exception, overwritten copies, and `-reset`

**What to build:** A file edited on the VM survives while the host's copy is unchanged, and is reported at each session start. When the host's copy changes, the host wins: the agent saves the VM's version under `.sync-vm/overwritten/` and the host prints an alert. `sync-vm -reset` replaces every kept VM edit.

**Blocked by:** 01, 02

**Status:** ready-for-agent

- [ ] Agent step 4 branches: no file → write; matching content → record as synced without writing (row 5 crash recovery); unrecorded or edited content → copy to `<root>/.sync-vm/overwritten/<path>` (replacing an earlier copy) and add to `Ack.overwrote`, then write.
- [ ] Agent step 6: delete acts only on recorded paths; an edited file is saved to `overwritten/` and reported first.
- [ ] Host prints `kept VM edit: <path>` once per path at session start only, and `replaced VM edit: ...` with the documented text for each `overwrote` path.
- [ ] `-reset` flag on the sync command passes `reset` to the first session's full `Diff` only.
- [ ] End-to-end: edit a file on the VM side, reconnect, edit is kept; change it on the host, it is replaced and reported; run with `-reset`, every VM edit is replaced.
- [ ] Agent tests: each branch of step 4, a delete of an edited file, and a crash simulated by writing a file without saving state.

---

## 05: Watch mode

**What to build:** After the initial sync, `sync-vm` keeps running and pushes host edits to the VM within about a second, filtering out ignored files, and tells the user when the VM is up to date.

**Blocked by:** 01

**Status:** ready-for-agent

- [ ] `files.Watch` with `quiet` 150 ms and `maxWait` 1 s; drops `.git/` events; sends `nil` on dropped events or any `.gitignore` in a batch; watches `<gitDir>/index` (outside root for worktrees) and sends `nil` at most once per 2 s, trailing; skips watches under ignored folders unless an allowlist pattern could match inside.
- [ ] `Tree.Stat`: known-set paths need no git command; other paths are filtered with the three `ls-files` commands using `:(literal)` pathspecs; more than 500 → falls back to `List`; deleted paths leave the known set. `List` replaces the known set.
- [ ] `engine.Pending` (`Add`, `Take`, capacity-1 signal channel; `nil` sets `all`).
- [ ] Session steps 1 and 4: discard pending before the full listing, then loop on `Pending` using `List` or `Stat`, `Diff` with `reset` false, no `kept` lines.
- [ ] Up-to-date reporting: `up to date (<n> files)` after the initial sync; `synced: <p> updated, <d> deleted` only when a counter is nonzero, omitting zero parts; reported when nothing is in flight and `Pending` is empty.
- [ ] Ctrl+C cancels and exits cleanly.
- [ ] Tests: a batch goes out within 1 s while a file changes every 100 ms; several index changes in 2 s give one `nil`; `.git/` events dropped; `.gitignore` change sends `nil`; `Stat` on a known path runs no git (counted); a new file under an ignored folder is dropped; a new untracked file is kept; >500 paths falls back to `List`; output lines appear exactly as specified.

---

## 06: Reconnect and safety guards

**What to build:** A dropped connection recovers on its own and converges without any resume logic. A failing or empty host listing never damages the VM.

**Blocked by:** 05

**Status:** ready-for-agent

- [ ] On disconnect print `disconnected, retrying`, back off from 1 s doubling to 30 s, reset after a session reaches the watch loop; each reconnect re-runs the full manifest exchange.
- [ ] `-reset` applies only to the first session.
- [ ] Failed operations from an `Ack` are removed from the picture and retried with the next batch with growing delays.
- [ ] Git error guard: print `git failed: <error>. Retrying in 2 seconds.`, send nothing, then `Add(nil)` after 2 s; session stays open.
- [ ] Never-delete-everything guard: empty `local` with a non-empty VM picture prints the documented warning and stops processing batches for that VM until restart.
- [ ] End-to-end: kill the connection mid-sync and the folders converge; a failing listing (e.g. `.git/index.lock` present) sends nothing and retries; an empty listing deletes nothing.

---

## 07: Agent durability and throughput

**What to build:** The agent survives crashes and power loss without mistaking a half-written file for a VM edit, never leaves temporary files in source folders, and keeps up with large first syncs without rewriting `state.json` constantly. The host pipelines operations.

**Blocked by:** 01

**Status:** ready-for-agent

- [ ] Reader goroutine plus apply loop; `Ack` after 32 operations, after 50 ms with at least one applied, or when the queue empties.
- [ ] Save at most once per second, plus after 200 ms idle and at end of input.
- [ ] Durable save order: flush files (`unix.Syncfs` on Linux; `File.Sync` per unflushed path on macOS/Windows), write and sync `state.json.tmp`, rename, sync the `.sync-vm/` folder on Linux/macOS.
- [ ] Agent "current hash" semantics and the 2-second rule for every file it hashes or writes.
- [ ] Temporary files under `<root>/.sync-vm/tmp/<seq>.tmp`, cleared at session start; cross-filesystem fallback to `.<name>.<seq>.tmp` beside the target; brief retry on Windows when the target is in use, then report failed.
- [ ] Host keeps up to ~64 operations in flight.
- [ ] Tests with a fake clock: each `Ack` trigger; 10,000 puts save at most once per simulated second. With a recording fake filesystem: the exact save order. Simulated power loss (state kept, file truncated) leads the host to resend, not keep. Leftover tmp file deleted at next session; temp files never created beside targets.

---

## 08: Host hash cache and fast startup

**What to build:** A normal `sync-vm` start or rescan reads only files that changed since the last run, even on a fresh process, by trusting size and modified time like `git status` does. Files that do need hashing are hashed in parallel.

**Blocked by:** 01

**Status:** ready-for-agent

- [ ] In-memory cache `{size, modTime, hash}` per path, never written to disk; size and modified time from `os.ReadDir` / `DirEntry.Info`, not a per-file stat.
- [ ] `Tree.Seed` adds records from manifest `HostSize`/`HostModTime`/`Synced` only for paths without a record and with nonzero `HostModTime`; engine calls it before `List`.
- [ ] 2-second rule on the host: zero sub-second part and under 2 s old → record and `Entry.ModTime` set to zero.
- [ ] Parallel hashing with `min(8, NumCPU)` workers on the host, and the same pool in agent manifest preparation.
- [ ] Concurrent `List` calls share one run via `singleflight`; `Tree` is safe for concurrent use.
- [ ] Tests with a counting file opener: `List` doesn't open a file whose size and modTime match a cache or `Seed` record; 2-second rule with an injected clock; two concurrent `List`s run git and hash each file once; race detector on. End-to-end: restart with a new `Tree`, change one host file, only that file is opened.

---

## 09: Copy instead of resending

**What to build:** Renaming a folder, switching branches, or syncing duplicate files makes the agent copy content it already holds instead of the host resending it. A VM edit to a copy source never spreads.

**Blocked by:** 01

**Status:** ready-for-agent

- [ ] `Copy` message round-trips in `protocol`.
- [ ] Pure `PlanCopies` per the doc: source must have `Hash == Synced` and match the put's hash; lowest path wins; puts stay puts for `noCopy` paths, case-only renames, and file↔folder changes; order is copies, deletes, puts.
- [ ] Agent step 5: fail with `source changed` if the source record is missing or its current hash differs; otherwise apply step 4 with content from the source.
- [ ] Engine adds failed copy paths to a per-session `noCopy` so the retry sends content.
- [ ] Unit tests for every `PlanCopies` case listed in the build order. End-to-end: rename a folder of 1,000 files with no host file opened and only copies/deletes sent; edit a copy source on the VM side first, and the failed copy is retried as a put.

---

## 10: Allowlist and listing filters

**What to build:** A user can opt gitignored files such as `.env` into syncing with `sync-vm allowlist`, and the host listing safely skips things that can't or shouldn't sync, with one clear warning each.

**Blocked by:** 02

**Status:** ready-for-agent

- [ ] `allowlist add`: validates each pattern with `git ls-files -oi --exclude-standard -z -- ':(glob)<pattern>'`, saves nothing and prints git's message on error, ignores duplicates, prints each pattern with its current match count (noting a pattern that matches nothing yet).
- [ ] `allowlist remove`: an unsaved pattern is an error and nothing is removed. `allowlist list` prints patterns or `The allowlist is empty.`
- [ ] `Tree` takes the allowlist at start; `List` runs the third `ls-files` command only when the allowlist is non-empty; `Stat` keeps a new path from that command only if it was in the batch.
- [ ] Skips with a warning, once per path per process: submodules (`160000`), symlinks (`120000`), names `RelPath` rejects, case-only collisions. Tracked files missing from disk are skipped silently.
- [ ] Untracked and allowlisted names on the editor temporary-file list are skipped silently; tracked files are never skipped by it.
- [ ] Tests: allowlist add with new, duplicate, and git-rejected patterns; remove with saved and unsaved; list; ignored `.env` included only when a pattern matches; each editor temp name skipped untracked, kept tracked; submodule and symlink skipped with a warning.

---

## 11: Executable bit and CRLF warning

**What to build:** Scripts marked executable in git (e.g. `gradlew`) run on Linux and macOS VMs, flipping the mark syncs without resending content, and a Windows host with CRLF scripts gets a clear warning with the fix.

**Blocked by:** 01

**Status:** ready-for-agent

- [ ] `Entry.Exec` from mode `100755`; for untracked/allowlisted files, the owner-execute bit on Linux/macOS hosts, false on Windows hosts.
- [ ] `Put` carries `exec`; agent records it; permissions rule `0755`/`0644` on Linux/macOS, applied at temp-file creation; ignored but recorded on Windows.
- [ ] `Diff` rows 4 and 7 compare `Exec`; a flag-only change becomes a put, and the agent only calls `os.Chmod` (step 4.2). Counted as "updated".
- [ ] `Tree.CRLFScripts` from `git ls-files --eol -z`: `w/crlf` and (`Exec` or `.sh`).
- [ ] Warning printed once per Linux/macOS VM on the first successful connect, computed once per process, at most 5 names then `and <n> more`, with the `.gitattributes` fix text from the doc. Syncing continues.
- [ ] Tests: `Diff` cases where only `Exec` differs (rows 4 and 7); a put with `exec` true creates `0755`; a flag-only put chmods without writing; `CRLFScripts` against a `core.autocrlf=true` repo with a `.sh`, an executable, and a plain text file.

---

## 12: Several VMs in one process

**What to build:** One `sync-vm` run keeps every selected VM in sync at once, reading and hashing each changed host file once. A slow, broken, or refused VM affects only itself.

**Blocked by:** 05, 06

**Status:** ready-for-agent

- [ ] One `Tree`, one shared auth store (placeholder until ticket 14), one `Watch`; one `Pending` per target receiving every batch; one `engine.Sync` goroutine per target.
- [ ] Every line from a target's `Sync` is prefixed `[<target>] `.
- [ ] Ctrl+C cancels all and waits; exit status 0 only if no `Sync` returned an error.
- [ ] End-to-end with two in-process agents: kill one connection and the other keeps converging; each changed host file is opened once, not once per VM; a `Fatal` on one VM leaves the other running.

---

## 13: SSH bootstrap to a real VM

**What to build:** `sync-vm init linux` and `sync-vm` work against a real VM over a built-in SSH client: no `ssh.exe`, no manual agent install. The right agent build is detected, uploaded once per version, and started over the SSH session's stdin/stdout.

**Blocked by:** 03

**Status:** ready-for-agent

- [ ] `scripts/build-dist.sh` builds agents for linux/darwin/windows × amd64/arm64 into the embedded agents folder, then builds `sync-vm` with the version `-ldflags`.
- [ ] Target resolution from `~/.ssh/config` via `kevinburke/ssh_config`: `HostName`, `User` (target's `user@` wins), `Port`, repeated `IdentityFile` with `~` expansion and documented defaults.
- [ ] Key login with unencrypted keys via `golang.org/x/crypto/ssh`; unreadable/unsupported keys (`.ppk`, `sk-`) skipped with a warning.
- [ ] Host key check with `skeema/knownhosts`: `HostKeyAlgorithms` set from known entries; known-match continues; changed key fails without retry and prints the documented warning; unknown key prompts with fingerprint and appends on `y`/`yes` (creating `~/.ssh` and the file, `0600` on Linux/macOS).
- [ ] TCP dial with a 10 s timeout; one `ssh.Client` per connection.
- [ ] Platform detection with `uname -sm`, falling back to `cmd /c echo %PROCESSOR_ARCHITECTURE%`; arch mapping as documented.
- [ ] SFTP install to `.sync-vm/agent/<16 hex of SHA-256>/sync-vm-agent[.exe]`: skip if present, else `MkdirAll`, write `.tmp`, chmod `0755` on Unix, rename (posix-rename if supported). No SFTP → documented failure.
- [ ] Start the agent with the per-OS run command; closing the stream closes stdin first, then session and client; stderr printed with `-v`.
- [ ] `Dial` errors say whether retrying can help.
- [ ] Agent step 0: touch own folder, remove sibling builds older than 7 days in a goroutine, ignoring errors.
- [ ] `-v` logs each connection, remote command, put/delete with size, `Ack`, and listing/batch timings.
- [ ] Tests against an in-process SSH + SFTP server: unknown key prompt yes/no, changed key fails without prompt, Ed25519-only `known_hosts` still connects to a server offering RSA, install then second dial skips the upload. Agent test: step 0 removes old siblings and keeps newer ones and its own.
- [ ] Manual check: `init` and sync against a real Linux VM.

---

## 14: SSH login prompts

**What to build:** Passphrase-protected keys, password-only VMs, and keyboard-interactive logins work, with each secret asked once per run and reused across every VM and reconnect. Prompts never overlap and never hang without a terminal.

**Blocked by:** 13

**Status:** ready-for-agent

- [ ] `bootstrap.NewAuth` store: unlocked keys by file path, passwords by `user@host:port`, one prompt mutex; secrets never written or logged.
- [ ] Passphrase prompt `Passphrase for <file>: ` on `PassphraseMissingError`, up to 3 tries, then skip the key.
- [ ] Password method with prompt `Password for <user>@<host>: `; keyboard-interactive answers hidden `password` questions with the same password and prompts every time for other questions.
- [ ] Passwords stored after a successful login; a failed login forgets that target's password.
- [ ] Every prompt takes the mutex and is prefixed `[<target>] `; uses `term.ReadPassword`; without a terminal, fails that VM with the documented message.
- [ ] Failed login is not retryable.
- [ ] Tests with a counting fake prompter: two dials plus a reconnect with one passphrase key prompt once; wrong passphrase re-asked up to 3 times; password-only and keyboard-interactive servers prompt once per run; a rejected stored password is asked again; no-terminal failure message.

---

## 15: SSH connection robustness

**What to build:** VMs behind a jump host and SSH configs that use `Include` work, unsupported settings are visible with `-v`, and a VM that sleeps or drops off the network is detected and reconnected without prompting.

**Blocked by:** 13

**Status:** ready-for-agent

- [ ] `ProxyJump` with one hop, written as an alias or `[user@]host[:port]`, resolved with the same rules; VM connection opened through the jump client.
- [ ] `Include` honored; `Match` ignored.
- [ ] With `-v`, log the name of each ignored setting that applies to the host.
- [ ] Keepalive goroutine: `keepalive@openssh.com` every 15 s with reply; 3 consecutive failures or 15 s timeouts close the client, triggering the engine's reconnect.
- [ ] Tests against the in-process SSH server: `ProxyJump`, `Include`, `user@` overriding `User`; a server that stops answering is closed within 60 seconds.
