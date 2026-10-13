# sync-vm Sync Architecture

Handoff for the redesign of `sync-vm`. This replaces the transport, snapshot, and workflow sections of `docs/architecture-and-workflow.md`. Nothing here is implemented yet.

## Goal

A developer sets up each VM once from inside a git repo, the way `git init` sets up a repository:

```bash
sync-vm init linux
sync-vm init win:C:\dev\proj
```

After that, one command keeps every set-up VM in sync:

```bash
sync-vm
```

`sync-vm` detects each VM's OS, installs the right agent if needed, and keeps each VM folder matching the repo until stopped. No port tunnels, no manual agent install, no manual upgrades, and one SSH passphrase prompt per run.

The rule the whole design serves is that every file in a VM folder converges to the host's version, whatever happened before, including crashes and dropped connections. There is one exception. If a synced file was edited on the VM and the host's copy has not changed since the last sync, the VM's edit stays. It stays until the host's copy changes or the user runs `sync-vm -reset`.

The exception exists for build tools that edit tracked files on the VM. In the main workflow, the user edits on the host and builds on the VM. `yalc add` rewrites `package.json`, and `yarn install` can rewrite `yarn.lock`. Without the exception, every `sync-vm` start would revert those edits.

`sync-vm` never writes into the host repo's working tree. Its only host files are `sync-vm.json` inside the repo's git folder, which git never commits or pushes, and new lines in `~/.ssh/known_hosts` when the user trusts a new VM.

## Set up SSH access first

`sync-vm` makes its own SSH connections. It doesn't run `ssh.exe` and doesn't use ssh-agent. It reads the same files `ssh` does, all from the user's own folder, so it needs no admin rights.

1. Give each VM a Host alias in `~/.ssh/config` (`C:\Users\<you>\.ssh\config` on Windows). The alias is the name used with `sync-vm`. If the VM's address changes, edit only `HostName`.

	```
	Host linux
		HostName 192.168.1.50
		User me
		IdentityFile ~/.ssh/id_ed25519

	Host win
		HostName 192.168.1.51
		User me
		IdentityFile ~/.ssh/id_ed25519
	```

2. Put the public key, `~/.ssh/id_ed25519.pub`, on each VM. On Linux and macOS, add it to `~/.ssh/authorized_keys`. On Windows, add it to `C:\Users\<you>\.ssh\authorized_keys`, or to `C:\ProgramData\ssh\administrators_authorized_keys` if the VM account is an administrator. Windows OpenSSH ignores the per-user file for administrators.

The key may have a passphrase. `sync-vm` asks for it once per run and keeps the unlocked key in memory until it exits. VMs that use the same key share the one prompt. If a VM accepts only passwords, `sync-vm` asks for that VM's password once per run instead. The first connection to a new VM shows its host key fingerprint and asks whether to trust it. See "bootstrap" for the details.

## Speed tips (not handled by sync-vm)

These settings belong to the user's machines, git, and security policy. `sync-vm` never changes them, never checks them, and works without them. Each one trades some protection or setup effort for speed, so the user, or whoever sets the security policy, decides. None of the speedups they give has been measured for `sync-vm`.

- **Exclude the VM folder from Windows Defender on a Windows VM.** Defender scans each file as it is created, which makes a first sync of many small files slow. In an administrator PowerShell on the VM, run `Add-MpPreference -ExclusionPath 'C:\dev\proj'`. Files in an excluded folder are no longer scanned, so check that the company policy allows it.
- **Exclude the repo from Windows Defender on the host.** The same scan runs each time the host opens a file to hash or send it. Run the same command on the host with the repo's path. The same policy question applies.
- **Turn on git's untracked cache and file system monitor on the host.** Run `git config core.untrackedCache true` and `git config core.fsmonitor true` in the repo. These make git's scans for untracked files faster in large repos. `sync-vm` runs that kind of scan on every full listing. Whether `git ls-files` benefits as much as `git status` should be measured on the real repo.

## Decisions already made

| Question | Answer |
|---|---|
| Sync direction | Host to VM only. No file content ever flows from a VM to the host. |
| Who wins a conflict | The host. A VM edit survives only while the host's copy is unchanged since the last sync. |
| A VM edit is replaced | The agent saves the VM's version under `<dest>/.sync-vm/overwritten/`, and the host prints an alert naming the file. |
| Discard all VM edits | `sync-vm -reset` |
| What can be synced | Only a git repo. `sync-vm` refuses to run outside one. Gitignored files sync only when they match the repo's allowlist. |
| Where the VMs and folders are set | `sync-vm init <target>[:<path>]`, once per VM. The result is saved in `<git-dir>/sync-vm.json`. |
| Host files `sync-vm` writes | Only `<git-dir>/sync-vm.json`, and `~/.ssh/known_hosts` when the user trusts a new VM. Nothing in the working tree, so nothing `sync-vm` writes can be committed or pushed. |
| Several VMs for one repo | One `sync-vm` process syncs to every saved VM at once, sharing one file listing and one watcher. |
| Protect against a wrong destination | A destination must be set up once with `sync-vm init`, which accepts only a new or empty folder. `-force` also accepts a folder that already has files or belongs to another repo. A normal run accepts only a folder that this repo set up. Every run rejects relative paths and a fixed list of system folders. See "Check the destination". |
| SSH client | Built in, using `golang.org/x/crypto/ssh`. No `ssh.exe`, `scp`, or ssh-agent. Each key's passphrase is asked once per run. |
| VM OS and architecture | Auto-detected over SSH |
| Agent distribution | Every agent build is embedded in `sync-vm` (estimated ~30 MB total) |
| Transport | One SSH connection per VM. The agent's stdin and stdout carry the protocol. The TCP tunnel mode is removed. |
| Dependencies | Company policy. Only the Go standard library, `golang.org/x/...` modules, and `github.com/fsnotify/fsnotify`. `~/.ssh/config` parsing, the `known_hosts` algorithm lookup, and the SFTP client are written in this repo. A test fails the build if `go.mod` requires any other module. |

## Commands

Every command except help and version runs from inside a git repo. `sync-vm` finds the repo with `git rev-parse --show-toplevel` and its git folder with `git rev-parse --absolute-git-dir`. If either fails, it exits with `sync-vm must run inside a git repository.`

Commands that manage saved settings are grouped by what they manage: `vm` for VMs and `allowlist` for allowlist patterns. Inside a group, `list`, `add`, and `remove` always mean the same thing. `init` adds a VM, because adding one also means setting up its folder, so there is no `vm add`.

| Command | What it does |
|---|---|
| `sync-vm init <target>[:<path>] [-force]` | Sets up a folder on one VM and adds the VM to this repo's saved list. Syncs nothing. |
| `sync-vm [<target>...] [-reset]` | Syncs to every saved VM, or only to the named ones, until stopped with Ctrl+C. |
| `sync-vm vm list` | Prints the saved VMs and their folders. Doesn't connect to any VM. |
| `sync-vm vm remove <target>` | Removes one VM from the saved list. Doesn't connect to the VM, and leaves its folder as it is. |
| `sync-vm allowlist list` | Prints the allowlist patterns. |
| `sync-vm allowlist add <pattern>...` | Adds patterns for gitignored files that should sync anyway. |
| `sync-vm allowlist remove <pattern>...` | Removes allowlist patterns. |
| `sync-vm -h`, `sync-vm <command> -h` | Prints usage and exits with status 0. Doesn't need a git repo. |
| `sync-vm -version` | Prints the version and exits with status 0. Doesn't need a git repo. |

### Flags

| Flag | Accepted by | Meaning |
|---|---|---|
| `-force` | `init` | Set up a folder that already has files, or take over a folder owned by another repo. See check 5 under "Check the destination". |
| `-reset` | sync | Replace every kept VM edit on the selected VMs with the host's version. |
| `-v` | sync, `init` | Print detailed logs to stderr: each SSH connection and remote command, each put and delete with its size, each `Ack`, and how long each listing and batch took. |
| `-h`, `-help` | every command | Print usage for that command, or for `sync-vm` as a whole when no command is given. |
| `-version` | `sync-vm` with no command | Print `sync-vm <version> (protocol <n>)`. `<version>` is set at build time with `-ldflags "-X main.version=<version>"` in `scripts/build-dist.sh`, and is `dev` otherwise. `<n>` is the protocol version sent in `Hello`. |

Every flag also accepts two leading dashes, such as `--help` or `--force`.

Parse arguments in this order:

1. Split the arguments after `sync-vm` into flags and words. An argument that starts with `-` is a flag. Every other argument is a word. Flags can appear anywhere, so `sync-vm win -reset` and `sync-vm -reset win` mean the same thing. An argument of exactly `--` ends flag parsing. Every argument after it is a word, even if it starts with `-`. No flag takes a value, so a flag never consumes the next argument.
2. If `-h` or `-help` is present, print usage and exit with status 0, ignoring every other argument. Otherwise, if `-version` is present, print the version and exit with status 0.
3. If the first word is `init`, it is the command. If the first word is `vm` or `allowlist`, it names the group, and the second word must be one of that group's subcommands: `list` or `remove` for `vm`, and `list`, `add`, or `remove` for `allowlist`. A missing or unknown subcommand is an error. Otherwise, there is no command word, and every word is a target for the sync command. `init` therefore refuses `init`, `vm`, and `allowlist` as target names.
4. Parse the flags with a `flag.FlagSet` for that command, created with `flag.ContinueOnError`. A flag the command doesn't accept, such as `-reset` with `init`, is an error. On any error, print the message and that command's usage to stderr, and exit with status 2.
5. Check the number of remaining words. `init` and `vm remove` take exactly one. `vm list` and `allowlist list` take none. `allowlist add` and `allowlist remove` take at least one. The sync command takes any number.

Usage for `sync-vm` as a whole:

```
Usage:
	sync-vm init <target>[:<path>] [-force]   set up a folder on a VM
	sync-vm [<target>...] [-reset]            sync to every saved VM, or the named ones
	sync-vm vm list                           show saved VMs
	sync-vm vm remove <target>                forget a saved VM
	sync-vm allowlist list                    show allowlist patterns
	sync-vm allowlist add <pattern>...        sync gitignored files that match
	sync-vm allowlist remove <pattern>...     stop syncing them

Flags:
	-force     init: use a folder that already has files or belongs to another repo
	-reset     sync: replace every kept VM edit with the host's version
	-v         sync, init: print detailed logs
	-h         show help
	-version   show the version

Run sync-vm <command> -h for details on one command.
```

Each command's usage repeats its line from above, then describes its arguments and flags in one or two sentences each, using the wording of the tables in this section.

### Targets and paths

`<target>` is a Host alias from `~/.ssh/config`, or `[user@]host`. It is also the VM's name in the saved list and in output. Split the `init` argument at the first `:`. Everything before it is the target, and everything after it is the path. A Windows path keeps working, because `win:C:\dev\proj` splits into `win` and `C:\dev\proj`. An IPv6 address can't be written as a target. Use an SSH config alias for it.

If the path is left out, it is `~/<repo folder name>`, where the repo folder name is the last segment of `git rev-parse --show-toplevel`. The agent expands `~` (see "Check the destination").

### init

1. Parse the argument. If the target is `init`, `vm`, or `allowlist`, exit with `<target> is a sync-vm command and can't be a target name. Add a different Host alias for this VM in ~/.ssh/config.` If the saved list already has this target with a different folder, exit with `<target> is already set up for this repo at <root>. Run sync-vm vm remove <target> first.`
2. If the repo has no `sync-vm.json` yet, create one with a new repo ID.
3. `bootstrap.Dial` the target. Send `Hello` with `init` true, `force` from `-force`, the path, and the repo ID.
4. On `Fatal`, print the reason and exit with a nonzero status. Save nothing.
5. On `Manifest`, save the target with `Manifest.root`, the absolute path the agent resolved. Print `<target>:<root> is ready. Run sync-vm to start syncing.` and exit with status 0.

Rerunning `init` for a target that is already saved with the same folder is safe. It changes nothing.

### vm list

Print one line per saved VM, in the order they were added: the target, two spaces, then the root.

```
linux  /home/me/proj
win  C:\dev\proj
```

If the list is empty, print `No VMs are set up for this repo. Run sync-vm init <target>.` Exit with status 0 in both cases. `vm list` reads only `sync-vm.json`, so it doesn't show whether a VM is reachable or its folder still exists.

### vm remove

Delete the target from the saved list and print `Removed <target>. The folder <root> on <target> is left as it is. Delete it on the VM if you no longer need it.` If the target isn't saved, exit with an error that lists the saved targets.

### allowlist

Git skips gitignored files, so files like `.env` or `appsettings.Development.json` never reach the VM. The allowlist holds patterns for gitignored files that should sync anyway. The patterns are saved in `sync-vm.json`, so they are never committed, and they apply to every VM of the repo.

A pattern is a git glob pathspec, relative to the repo root, as used by `git ls-files -- ':(glob)<pattern>'`. `*` matches within one path segment, and `**` matches across segments. Examples are `.env`, `**/.env`, and `config/*.local.json`.

- `sync-vm allowlist add <pattern>...` adds each pattern that isn't saved yet, and ignores one that is. It first checks each pattern by running `git ls-files -oi --exclude-standard -z -- ':(glob)<pattern>'`. If git reports an error, it saves nothing and prints git's message. Then it prints each pattern with the number of files it matches now. A pattern that matches no file is still saved, with a note that it matches nothing yet.
- `sync-vm allowlist remove <pattern>...` removes each pattern. A pattern that isn't saved is an error, and nothing is removed.
- `sync-vm allowlist list` prints the saved patterns, one per line, or `The allowlist is empty.`

A running `sync-vm` reads the allowlist only when it starts. To apply a change, restart it. A file that stops matching after `allowlist remove` is deleted from each VM on the next sync, like any other file the host no longer has.

The allowlist only adds gitignored files. To stop syncing a tracked file, use git, for example by deleting it or adding it to `.gitignore` and running `git rm --cached`.

### sync (no command word)

1. Load `sync-vm.json`. If it lists no targets, exit with `No VMs are set up for this repo. Run sync-vm init <target>.`
2. With no target arguments, select every saved target. With arguments, select only those. If an argument isn't saved, exit with an error that lists the saved targets and suggests `sync-vm init <target>`.
3. Run the selected targets together, as described in "Syncing several VMs".

## Core idea: compare state, don't replay events

The host never trusts its memory of a VM. Each session starts with the agent listing what it holds. For each file, the listing gives two hashes: the content on disk now, and the content the agent last wrote from the host. The host compares those with its own files and sends the difference. Watch mode runs the same comparison over only the paths that changed. The first sync, a reconnect, and a watch batch all use one code path.

The second hash is what lets the host tell who changed a file. If the VM's content differs from what the agent last wrote, the VM edited it. If the host's content differs from what the agent last wrote, the host edited it. The host keeps no record of file state between runs. The agent's record is the only one.

File watcher events are hints that something may have changed. They are never treated as a record of what changed.

## Data

- **`RelPath`.** A slash-separated path relative to the sync root. Only a parser can create one. It rejects anything unsafe or invalid on either Windows or Linux: empty segments, `.` and `..`, a leading `/`, `\`, `:` (drive letters, NTFS streams), `* ? " < > |`, control characters, segments ending in a dot or space, and reserved Windows names (`CON`, `PRN`, `AUX`, `NUL`, `COM1` to `COM9`, `LPT1` to `LPT9`, with or without an extension). It also rejects a first segment equal to `.sync-vm` in any letter case, because the agent keeps its own files there. Both sides parse paths when a message is decoded. Code past that point trusts them.
- **`Hash`.** SHA-256 of a file's content, sent on the wire as 64 lowercase hex characters.
- **`ModTime`.** A file's last-modified time as Unix nanoseconds (`int64`). Zero means unknown. Zero never matches a real modified time.
- **`RepoID`.** 32 lowercase hex characters from `crypto/rand`, created once per repo by the first `init`. It ties a VM folder to the host repo that set it up. A git worktree has its own git folder, so it gets its own ID.
- **`Entry`.** `{Path, Size, ModTime, Hash, Exec}`. A host file. `ModTime` is the modified time the host saw when it hashed the file, or zero under the 2-second rule in the files section. `Exec` is true when the file should be executable on a Linux or macOS VM (see files). Only files are synced. Directories exist on a VM only as parents of synced files.
- **`RemoteEntry`.** `{Path, Size, Hash, Synced, HostSize, HostModTime, Exec}`. `Exec` is the executable flag from the last `Put` the agent applied to that path. A VM file the agent holds. `Size` and `Hash` describe the content on disk now. `Synced` is the hash of the content the agent last wrote to that path from a `Put`. When `Hash` equals `Synced`, the file is unedited on the VM. When they differ, someone on the VM edited it. `HostSize` and `HostModTime` are the size and modified time of the host's file when the host sent that `Put`. The host uses them to skip hashing on a fresh start (see files).
- **`Change`.** `{Path, Delete bool}`.
- **Repo config.** `<git-dir>/sync-vm.json`, written only by `init`, `vm remove`, `allowlist add`, and `allowlist remove`, and read by every command except help and version. Write it to a temporary file in the same folder, then rename it over the old one.

	```json
	{
		"repo": "3f9a0c1d2b7e4f6a8c5d9e0b1a2f3c4d",
		"vms": [
			{"target": "linux", "root": "/home/me/proj"},
			{"target": "win", "root": "C:\\dev\\proj"}
		],
		"allowlist": [".env", "config/*.local.json"]
	}
	```

	`root` is the absolute path the agent resolved during `init`. It never contains `~`. `allowlist` holds the allowlist patterns, in the order they were added. A missing `allowlist` means no patterns.
- **Host session state.** Kept in memory per VM and discarded on disconnect. It holds the host's picture of that VM's files as `map[RelPath]RemoteEntry`, the next sequence number, and the operations sent but not yet acknowledged.
- **Agent state.** `<dest>/.sync-vm/state.json` on the VM: `{"repo": RepoID, "files": {RelPath: record}}`. Each record describes one file the agent wrote: `{size, modTime, hash, synced, hostSize, hostModTime, exec}`. `size`, `modTime`, and `hash` describe the file on disk when the agent last looked. `synced` is the hash of the content it last wrote from a `Put`. `hostSize`, `hostModTime`, and `exec` are copied from that `Put`. The file is on the VM's disk, so it survives agent restarts, reconnects, and VM reboots. The agent is the only writer of this file. It saves the file as described in "Save state durably" under agent, which keeps a power loss from making a half-written file look like a VM edit.

## Wire protocol

Each frame is a 4-byte big-endian length, a 1-byte tag (`M` for message, `D` for data), then the payload. A message payload is JSON `{"kind": ..., "body": ...}`. One frame is capped at 64 MB. File content travels as 1 MB data frames, so there is no file size limit.

| Message | Direction | Meaning |
|---|---|---|
| `Hello{version, root, repo, init, force}` | host to VM | Opens the session. For `init`, `root` is the path from the command line or the default, and may start with `~`. For a normal run, it is the saved absolute `root`. `repo` is the repo ID. `init` is true for `sync-vm init`. `force` is true when the user passed `-force` to `init`. |
| `Manifest{root, files []RemoteEntry, more}` | VM to host | `root` is the absolute path the agent resolved. `files` lists files the agent wrote and still holds. The agent sends at most 10,000 files per message, with `more` true on every message except the last. The host collects messages until `more` is false. A single message would pass the 64 MB frame cap at about 250,000 files. `files` is empty for `init`. |
| `Put{seq, path, size, hash, modTime, exec}`, then data frames | host to VM | Make the file at `path` hold exactly these `size` bytes. `modTime` is the host file's `Entry.ModTime`. `exec` is `Entry.Exec`. |
| `Copy{seq, path, from, size, hash, modTime, exec}` | host to VM | Make the file at `path` a copy of the VM's own file at `from`, which must hold content with hash `hash`. No data frames follow. The other fields mean the same as in `Put`. See "Copy instead of resending" under engine. |
| `Delete{seq, path}` | host to VM | Remove one file, and any parent folders it leaves empty |
| `Ack{through, failed []{seq, path, reason}, overwrote []path}` | VM to host | Every operation up to `through` is handled. Those listed in `failed` were not applied. `overwrote` lists the paths where a `Put`, `Copy`, or `Delete` replaced content the agent did not write. |
| `Fatal{reason}` | VM to host | The agent refuses the session, for example because another host holds the folder |

`RENAME`, `MKDIR`, `RMDIR`, `SNAPSHOT_START`, and `SNAPSHOT_END` from the old design are gone. A rename is a delete plus a put, or a copy when the VM already has the content. Puts and copies create parent folders.

The host always sends the data frames for a `Put`, even when the agent ends up not writing them (see agent step 4). This keeps the framing simple. That case occurs only after a crash, so the wasted bytes don't matter.

## Modules

Two binaries, eight internal packages.

```
cmd/sync-vm/          host CLI: parse the command, then run init, a vm or allowlist subcommand, or sync
cmd/sync-vm-agent/    VM agent: agent.Serve over stdin and stdout
internal/protocol/    framing, message types, RelPath parsing
internal/repo/        find the repo and git folder, parse targets, load and save sync-vm.json
internal/bootstrap/   SSH connections and login, detect the VM platform, install and start the agent
internal/sshconfig/   parse ~/.ssh/config for the settings bootstrap supports
internal/sftp/        the SFTP v3 client subset bootstrap uses to install the agent
internal/files/       list, stat, and watch host files
internal/engine/      the host sync loop per VM, and the pure Diff function
internal/agent/       apply changes inside the VM root
```

### protocol

Owns the wire format. Exposes a `Conn` with `Send(Msg)` and `Recv() (Msg, error)` over any `io.ReadWriteCloser`. A received `Put` exposes its content as an `io.Reader` that reads the following data frames. Nothing outside this package sees frames or JSON. One goroutine may send while another receives.

### repo

- `Find(dir) (Repo, error)` runs `git rev-parse --show-toplevel` and `git rev-parse --absolute-git-dir` in `dir`. `Repo` holds the working-tree root and the git folder.
- `ParseTarget(arg, repoRoot) (target, path string, err error)` splits at the first `:` and fills in the default path as described in "Targets and paths".
- `(Repo).Load() (Config, error)` reads `sync-vm.json`. A missing file is an empty `Config` with no repo ID, no VMs, and no allowlist patterns.
- `(Repo).Save(Config) error` writes it as described under Data.

Nothing else reads or writes `sync-vm.json`.

### bootstrap

Owns every SSH connection. `sync-vm` doesn't run `ssh.exe` or `scp`, and doesn't use ssh-agent. This is what lets it ask for a passphrase once per run: it unlocks each key once and reuses it for every VM, every connection, and every reconnect. Running `ssh.exe` would mean one prompt per `ssh` process, and Windows OpenSSH has no connection sharing (`ControlMaster`) to avoid that.

Libraries:

| Library | Use |
|---|---|
| `golang.org/x/crypto/ssh` | SSH client, key parsing, sessions, and the `sftp` subsystem channel |
| `golang.org/x/crypto/ssh/knownhosts` | Checking host keys and writing new `known_hosts` lines. The algorithm lookup is built on it, as described in "Check the host key". |
| `golang.org/x/term` | Reading a passphrase or password without echo |
| `internal/sshconfig` | Reading `~/.ssh/config`, including `Include`. See "Parse the SSH config". |
| `internal/sftp` | Uploading the agent over the same connection. See "The SFTP client". |

`NewAuth() *Auth` creates the store of unlocked keys and passwords. One `Auth` serves the whole process and every VM in it. It holds:

- unlocked keys, by key file path
- passwords, by `user@host:port`
- one mutex that every prompt takes, so two VMs never prompt at the same time

Secrets stay in memory until the process exits. They are never written anywhere and never logged, including with `-v`.

`Dial(ctx, target, auth) (io.ReadWriteCloser, Platform, error)` connects to one VM and starts its agent. `Platform` is `{OS, Arch}`, with `OS` one of `linux`, `darwin`, or `windows`. The returned error says whether retrying can help. Network errors can be retried. A failed login, a refused or changed host key, and a missing SFTP subsystem can't.

#### Resolve the target

Split `target` into an optional user and a host at the last `@`. Look the host up in `~/.ssh/config` and read these settings:

| Setting | Default when missing |
|---|---|
| `HostName` | The host as written |
| `User` | The user from `target`, which also wins over the config. Otherwise the current OS user name. |
| `Port` | `22` |
| `IdentityFile` | `~/.ssh/id_ed25519`, `~/.ssh/id_ecdsa`, and `~/.ssh/id_rsa`, each only if it exists. `IdentityFile` may repeat. Expand a leading `~`. |
| `ProxyJump` | None. One hop is supported, written as an alias or as `[user@]host[:port]`, and resolved with this same table. |

Ignore every other setting. With `-v`, log the name of each ignored setting that applies to the host, so a user can see why a setting has no effect. `Match` blocks aren't supported and are ignored.

#### Parse the SSH config

`internal/sshconfig` reads the file the way OpenSSH does, for the settings above only. `Lookup(host) (Settings, []string ignored, error)` returns the five settings for one host, and the names of the other settings that applied to it, for the `-v` log.

- **Lines.** Trim whitespace. Skip empty lines and lines starting with `#`. A line is a keyword and its arguments, separated by whitespace or by one `=` with optional whitespace around it, so `Port 22` and `Port=22` are the same. Keywords ignore letter case. An argument may be wrapped in double quotes, which allows spaces inside it.
- **Blocks.** Lines before the first `Host` or `Match` apply to every host. `Host <pattern>...` starts a block that applies when the host matches at least one pattern and no pattern starting with `!`. In a pattern, `*` matches any run of characters and `?` matches one, ignoring letter case. The host matched is the one written in the target, not its `HostName`. `Match` starts a block that never applies, up to the next `Host` or `Match`.
- **First value wins.** For each keyword, the first value from an applying line is the one used, and later lines for that keyword are skipped. Put specific `Host` blocks before `Host *`, as with `ssh`. The exception is `IdentityFile`, which collects every value from every applying line, in order.
- **`Include`.** Each argument is a path or a glob. A relative path is relative to `~/.ssh`. Expand a leading `~`. Expand globs with `filepath.Glob`, which returns sorted names. Parse each file in place, as if its lines appeared at the `Include`, under the condition of the block the `Include` sits in. A missing file is skipped, as `ssh` does. Stop with an error after 16 nested levels, so a file that includes itself can't loop.
- **Tokens.** In `HostName` and `IdentityFile`, replace `%h` with the host, `%d` with the local home folder, `%u` with the local user name, `%r` with the remote user, and `%%` with `%`. Any other `%` token makes that value invalid. Skip it with a warning naming the token.
- **Errors.** A missing `~/.ssh/config` is an empty config. A `Port` that isn't a number from 1 to 65535 is an error that names the file and line.

A setting from an `Include`d file is no different from one in the main file. No other part of `sync-vm` reads `~/.ssh/config`.

#### Log in

Set `ssh.ClientConfig.Auth` to these methods, in this order:

1. **Keys.** `ssh.PublicKeysCallback` returns one signer per `IdentityFile`. For each file, use the unlocked key from `Auth` if there is one. Otherwise read the file and call `ssh.ParsePrivateKey`. If it returns `*ssh.PassphraseMissingError`, prompt `Passphrase for <file>: ` and call `ssh.ParsePrivateKeyWithPassphrase`. If the passphrase is wrong, prompt again, up to 3 tries in all. Store the unlocked key in `Auth`. A key file that can't be read or parsed is skipped, with a warning naming the file. PuTTY `.ppk` files and hardware-backed `sk-` keys aren't supported, and the warning says so.
2. **Password.** `ssh.PasswordCallback` returns the stored password for `user@host:port`, or prompts `Password for <user>@<host>: `.
3. **Keyboard-interactive.** Answer each hidden-input question whose text contains `password`, in any letter case, with the same password as method 2. Prompt the user for any other question every time, because it is likely a one-time code.

After `Dial` logs in, store each password it used in `Auth`. If a login fails, remove that target's stored password, so the next attempt prompts again instead of reusing a wrong one.

#### Prompts

Every prompt takes the `Auth` mutex first, and starts with `[<target>] ` so the user knows which VM is asking. Read input with `term.ReadPassword` on the standard input's file descriptor. If standard input isn't a terminal (`term.IsTerminal` is false), don't prompt. Fail that VM with `<target> needs a passphrase, password, or host key decision, but sync-vm isn't running in a terminal.`

In normal use, prompts appear only at startup: once per key, once per password-only VM, and once per new VM's host key. A reconnect reuses what `Auth` holds and prompts only if something changed, such as a password changed on the VM.

#### Check the host key

Load `~/.ssh/known_hosts` with `knownhosts.New` from `golang.org/x/crypto/ssh/knownhosts`. A missing file counts as empty. Look the VM up by `HostName` and `Port`, joined with `net.JoinHostPort`. With `ProxyJump`, look up the VM's own address, not the jump host's. Set `ssh.ClientConfig.HostKeyAlgorithms` to the algorithms of the keys already known for that address. Without that, a server can offer an RSA key while `known_hosts` holds its Ed25519 key, and the check fails even though the VM is genuine.

Find the known keys through the callback itself, not by parsing `known_hosts` again. The callback already handles hashed host names (`|1|...`), `[host]:port` entries, wildcard patterns, and `@revoked` lines, and a second parser would disagree with it on some of them.

1. Generate one Ed25519 public key with `crypto/ed25519` when the process starts. It is random, so it never appears in `known_hosts`.
2. Call the callback with the address and that key. It returns a `*knownhosts.KeyError`. Its `Want` list holds every key known for the address. An empty `Want` means the host is unknown.
3. Take each key's `Type()`, in order, without repeats. Replace `ssh-rsa` with `rsa-sha2-512`, `rsa-sha2-256`, and `ssh-rsa`, in that order, because a server signs with an RSA key under those three algorithm names.
4. If the host is unknown, leave `HostKeyAlgorithms` unset, so the server's preferred key is offered and shown in the trust prompt.

The `HostKeyCallback` passed to `ssh.ClientConfig` wraps the same callback and turns its result into the rows below. A `*knownhosts.KeyError` with an empty `Want` is "not known". One with a non-empty `Want` is "known, but with a different key". A `*knownhosts.RevokedError` fails that VM without retrying, and names the revoked key.

| Lookup result | Action |
|---|---|
| Known, and the key matches | Continue. |
| Known, but with a different key | Fail that VM without retrying. Print `WARNING: the host key for <host> has changed. Someone may be intercepting the connection. If the VM was rebuilt, remove its line from ~/.ssh/known_hosts and run sync-vm again.` |
| Not known | Prompt `The authenticity of host <host> can't be established. <type> key fingerprint is <ssh.FingerprintSHA256>. Trust it? [y/N] `. On `y` or `yes`, append `knownhosts.Line` for the address to `~/.ssh/known_hosts`, creating `~/.ssh` and the file if needed, with mode `0600` on Linux and macOS. On anything else, fail that VM without retrying. |

#### Connect and keep the connection alive

Dial TCP with a 10-second timeout. With `ProxyJump`, first connect to the jump host the same way, then open the VM's connection through it with the jump client's `Dial`. Create one `ssh.Client` per VM connection. Every step below runs on that client as a session or over SFTP, so none of them prompts.

Start a keepalive goroutine. Every 15 seconds, send the global request `keepalive@openssh.com` with `wantReply` true. If 3 requests in a row fail or get no reply within 15 seconds, close the client. Without this, a VM that sleeps or loses its network leaves a connection that never errors, and that VM's sync stalls forever. After the close, the agent stream returns an error, and `Sync` reconnects.

#### Start the agent

1. **Detect the platform.** Run `uname -sm` in a session. If it fails, run `cmd /c echo %PROCESSOR_ARCHITECTURE%` in a new session. The login shell is unknown (sh, cmd, or PowerShell), and each probe works in every shell of its OS family. Map `x86_64`/`AMD64` to amd64 and `aarch64`/`arm64`/`ARM64` to arm64.
2. **Pick the embedded build** `agents/<os>-<arch>/sync-vm-agent[.exe]`.
3. **Install if missing.** Open an SFTP client on the connection with `internal/sftp` (see "The SFTP client"). The install path is `.sync-vm/agent/<first 16 hex chars of the binary's SHA-256>/sync-vm-agent[.exe]`, relative to the SFTP start folder, which is the login user's home folder. If `Stat` finds it, skip to step 4. Otherwise:
	1. `MkdirAll` the folder.
	2. Write the binary to the same name plus `.tmp`.
	3. On Linux and macOS, `Chmod` it to `0755`.
	4. Rename it into place. Use `PosixRename` when the server supports `posix-rename@openssh.com`. Otherwise use `Remove` on the target, ignoring "not found", then `Rename`.

	If the server offers no SFTP subsystem, fail that VM with `<target> doesn't offer SFTP. Enable the sftp subsystem in the VM's sshd_config.` Linux OpenSSH and Windows OpenSSH both enable it by default.
4. **Start it** in a new session. Take `StdinPipe` and `StdoutPipe`, collect stderr in a buffer, and call `Start` with the run command below. Return a stream that reads stdout and writes stdin. Closing the stream closes stdin first, so the agent sees end of input and exits, then closes the session and the client. With `-v`, print the agent's stderr.

Remote commands per OS. Each session runs its command through the VM's login shell.

| Step | Linux / macOS | Windows (works from cmd and PowerShell) |
|---|---|---|
| Detect | `uname -sm` | `cmd /c echo %PROCESSOR_ARCHITECTURE%` |
| Run | `./.sync-vm/agent/<h>/sync-vm-agent` | `.\.sync-vm\agent\<h>\sync-vm-agent.exe` |

Avoid parentheses in Windows commands. PowerShell evaluates `(exit 0)` itself before `cmd` sees it.

Old agent builds pile up in `.sync-vm/agent/` as `sync-vm` is upgraded. The agent removes them itself. See agent step 0.

#### The SFTP client

`internal/sftp` implements only what the install step needs, from SFTP version 3 (`draft-ietf-secsh-filexfer-02`), the version OpenSSH's `sftp-server` speaks on Linux, macOS, and Windows. It uses SFTP and not shell commands, because the login shell is unknown. Sending a binary through PowerShell's input, or through `certutil` with cmd's 8,191-character command limit, is not reliable. SFTP never involves the shell.

`NewClient(*ssh.Client)` opens a session, calls `RequestSubsystem("sftp")`, and sends `SSH_FXP_INIT` with version 3. A refused subsystem returns the "doesn't offer SFTP" error from step 3. The `SSH_FXP_VERSION` reply lists the server's extensions. Remember whether it includes `posix-rename@openssh.com`.

Each packet is a 4-byte big-endian length, a 1-byte type, then the fields, with strings as a 4-byte length and the bytes. Every request carries a request ID. Send one request at a time and wait for its reply. The install runs once per `sync-vm` version per VM, so pipelining isn't worth the code.

| Method | Request | Success reply |
|---|---|---|
| `Stat(path)` | `SSH_FXP_STAT` | `SSH_FXP_ATTRS`. `SSH_FX_NO_SUCH_FILE` returns an error that `errors.Is(err, fs.ErrNotExist)` matches. |
| `MkdirAll(path)` | `SSH_FXP_STAT`, then `SSH_FXP_MKDIR` for each missing segment from the top | `SSH_FX_OK` |
| `WriteFile(path, r)` | `SSH_FXP_OPEN` with `WRITE`, `CREAT`, and `TRUNC`, then `SSH_FXP_WRITE` in 32 KB chunks with increasing offsets, then `SSH_FXP_CLOSE` | `SSH_FXP_HANDLE`, then `SSH_FX_OK` for each write and the close |
| `Chmod(path, mode)` | `SSH_FXP_SETSTAT` with only the permissions flag | `SSH_FX_OK` |
| `Remove(path)` | `SSH_FXP_REMOVE` | `SSH_FX_OK` |
| `Rename(from, to)` | `SSH_FXP_RENAME` | `SSH_FX_OK` |
| `PosixRename(from, to)` | `SSH_FXP_EXTENDED` named `posix-rename@openssh.com` | `SSH_FX_OK` |

Any other status becomes an error carrying the status code and the server's message. A reply with an unexpected request ID or type is a protocol error that closes the client. Paths use `/` on every OS. A relative path is relative to the login user's home folder, on Windows OpenSSH too. 32 KB writes stay well under every server's packet limit.

### files

`NewTree(root) *Tree` owns the host's view of the repo's working tree for the life of the `sync-vm` process. One `Tree` serves every VM, so it is safe for concurrent use. It survives reconnects, so a reconnect doesn't hash the repo again.

`NewTree` takes the allowlist patterns from `sync-vm.json` along with the root.

- `(*Tree).List() (map[RelPath]Entry, error)` returns every file that should sync. It runs these commands, all with `-z`:

	| Command | Gives |
	|---|---|
	| `git ls-files -s` | Tracked files, each with its mode |
	| `git ls-files -o --exclude-standard` | Untracked files that aren't ignored |
	| `git ls-files -oi --exclude-standard -- ':(glob)<pattern>'...` | Ignored files that match an allowlist pattern. Run it only when the allowlist isn't empty. |

	Skip a tracked file that is missing from disk, because it was deleted and not yet staged. Skip a tracked entry with mode `160000`, which is a submodule, with a warning. Skip mode `120000`, which is a symlink, with a warning. Skip any name `RelPath` rejects, with a warning. Skip paths that differ only in case, because they would collide on a Windows VM, with a warning. Print each kind of warning once per path per process, not on every rescan.

	Skip untracked and allowlisted files whose name matches the editor temporary-file list below, without a warning. Editors create these files beside the real file while saving, and remove them within moments. Syncing them would create and delete files on the VM on every save, and wake any dev server watching the VM folder. Tracked files are never skipped by this list. The list is built in and has no setting.

	| Editor | File name patterns |
	|---|---|
	| Vim | `*.swp`, `*.swo`, `*.swx`, `4913` |
	| Emacs | `.#*`, `#*#` |
	| JetBrains IDEs | `*___jb_tmp___`, `*___jb_old___` |
	| Microsoft Office | `~$*` |
	| LibreOffice | `.~lock.*#` |
	| Many editors | `*~` |

	If any command fails, `List` returns an error. It never returns a partial or empty result in place of an error. A failing `git ls-files`, for example while another git command holds `.git/index.lock`, must not look like an empty repo.

	When several callers run `List` at the same time, they share one run and its result, using `golang.org/x/sync/singleflight`. Several VMs starting together then list and hash the repo once. `List` also replaces the tree's *known set*: the paths in its result. `Stat` uses the known set, as described below.
- `(*Tree).Stat(paths) (map[RelPath]Entry, error)` returns entries for those paths that exist and should sync. A path missing from the result is treated as deleted. It filters the paths with the same rules as `List`, in two groups:
	- **Paths in the known set.** Their eligibility to sync can only change through `.gitignore`, the git index, or the allowlist. The watcher turns the first two into a full `List`, and the allowlist is read only at start. So these paths need only a directory lookup, and no git command. This is the common case: editing an existing file.
	- **Other paths, such as new files.** Run the three commands above with each path added as `':(literal)<path>'`, and keep only paths they return. For the allowlist command, keep a path only if it is in the batch. Then apply the editor temporary-file list. Add each path kept to the known set. If this group has more than 500 paths, call `List` instead, because the command line would get too long on Windows.

	Remove each deleted path from the known set. Without this filter, a file created under an ignored folder such as `bin/` or `node_modules/` would sync as soon as the watcher reported it. Each `git` process costs about 50 ms to start on Windows, so skipping git for known paths takes that delay off each save.
- `(*Tree).CRLFScripts() ([]RelPath, error)` returns the files that would break on a Linux or macOS VM because of Windows line endings. It runs `git ls-files --eol -z` and keeps each file whose working-tree column is `w/crlf` and that either has `Exec` true or a name ending in `.sh`.

`Entry.Exec` is true when `git ls-files -s` reports mode `100755`. For an untracked or allowlisted file, `Exec` is the file's owner-execute bit on a Linux or macOS host, and false on a Windows host, which has no such bit. On a Windows host, mark a script executable with `git update-index --chmod=+x <file>`, then commit.
- `(*Tree).Seed(remote map[RelPath]RemoteEntry)` adds a cache record `{HostSize, HostModTime, Synced}` for each remote path that has no cache record yet and a nonzero `HostModTime`. Paths that already have a cache record keep it. That record came either from hashing the host file in this process, or from another VM's manifest, and both are equally valid.
- `Watch(ctx, root, gitDir, quiet, maxWait)` sends batches of possibly-changed paths after `quiet` (150 ms) passes with no new events, or after `maxWait` (1 s) since the batch's first event, whichever comes first. Without `maxWait`, a file on the host that changes more often than every 150 ms, such as a log file, would keep the batch open, and nothing would sync. It drops events under `.git/`, because git writes there constantly and nothing there syncs. If the OS drops events, or a batch contains a file named `.gitignore` anywhere in the tree, it sends `nil`, meaning rescan everything. An edit to `.gitignore` can change which files sync without touching them, so only a full `List` sees it.

	It also watches the single file `<gitDir>/index`, which can be outside `root` for a worktree. A change to it sends `nil`, at most once every 2 seconds. When several changes arrive within 2 seconds, it sends one `nil` after the last of them. Commands such as `git rm --cached` and `git add -f` change which files sync by changing only the index, and a full `List` is the only way to see that. On Linux, fsnotify watches one folder at a time, so new subfolders need new watches. Don't add watches under a folder that `git check-ignore -q <folder>` reports as ignored, unless an allowlist pattern could match inside it. That keeps a large `node_modules/` from exhausting the inotify watch limit.

#### Make startup and rescans cheap

Hashing a file means opening and reading it. On a Windows host, opening a file is the expensive part, because antivirus scans it on open. Without the two measures below, every `sync-vm` start reads the whole repo, which can take minutes for a large monorepo. Implement both. They are not alternatives:

- The hash cache decides which files need reading. On a normal start, it skips almost all of them.
- Parallel hashing speeds up the files that still need reading. Those are every file on the first sync or after `state.json` is lost, plus each file that changed since the last run.

**1. Hash cache.** Make `List` and `Stat` fill `Entry.Hash` through an in-memory cache, `map[RelPath]{size, modTime, hash}`. If a file's size and modified time match its cache record, use the cached hash and don't open the file. Otherwise, hash the file and replace the record. Never write the cache to disk.

The cache gets its records from two sources:

- **Hashing in this process.** Reconnects and full rescans after a watcher overflow reuse these records. A rescan then costs one directory listing, plus a read of each file that changed.
- **The agents' records, through `Seed`.** These cover a fresh start, when the cache is empty. When the host sent a file, the agent recorded the host file's size and modified time, and the content's hash as `Synced`. If the host file still has that size and modified time, treat its hash as `Synced` without reading it. A fresh start then costs one directory listing, plus a read of each file that changed since the last run. Git makes the same assumption for `git status`.

Get size and modified time from the directory listing (`os.ReadDir` and `DirEntry.Info`), not from a `stat` per file. On Windows, the listing returns both without opening each file, so antivirus doesn't scan it.

A file can change twice within one modified-time tick and keep the same size. Its cache record would then hold the old hash. The risk is real only on filesystems with coarse timestamps, such as FAT (2-second ticks) or HFS+ (1-second ticks). NTFS, ext4, and APFS keep times to 100 ns or finer, so a second write in the same tick is not a practical risk. Apply the *2-second rule*: when the host hashes a file whose modified time has a zero sub-second part, and is less than 2 seconds before the moment of hashing, set the record's `modTime` and the `Entry.ModTime` to zero. Zero never matches, so the file is hashed again on the next lookup, and the agent stores zero as `hostModTime`. A zero sub-second part is the sign of a coarse filesystem. On a fine-grained filesystem, it happens by chance about once in a billion writes, and then only costs an extra hash. Git uses a similar rule for its index. The agent's own size and modified-time check has the same risk and follows the same rule (see agent).

Without the sub-second test, every file the agent wrote in the last run would be hashed again on the VM at the next start, because the agent's own writes always fall inside the 2-second window.

**2. Parallel hashing.** Hash cache misses with a pool of `min(8, runtime.NumCPU())` workers, not one file at a time. The antivirus delay is per file, so it overlaps well across workers. Use the same pool in the agent's step 3, where the agent hashes again any file whose size or modified time changed.

### engine

`Diff(paths []RelPath, local map[RelPath]Entry, remote map[RelPath]RemoteEntry, reset bool) (changes []Change, kept []RelPath)` is pure. For each path, the first matching row decides:

| # | Host (`local`) | VM (`remote`) | Result |
|---|---|---|---|
| 1 | absent | absent | nothing |
| 2 | absent | present | delete |
| 3 | present | absent | put |
| 4 | `Hash` equals `remote.Hash`, `remote.Hash` equals `remote.Synced`, and `Exec` equals `remote.Exec` | | nothing |
| 5 | `Hash` equals `remote.Hash`, and `remote.Hash` differs from `remote.Synced` | | put (see below) |
| 6 | | `remote.Hash` equals `remote.Synced` | put |
| 7 | `Hash` equals `remote.Synced`, `Exec` equals `remote.Exec`, and `reset` is false | `remote.Hash` differs from `remote.Synced` | nothing, and add the path to `kept` |
| 8 | anything else | | put |

- Row 5 happens when the agent wrote a file but crashed before saving state. The put makes the agent record the content as synced. The agent sees the content already matches, so it writes nothing and reports nothing.
- Row 7 is the VM-edit exception: the VM edited the file and the host has not changed it since the last sync.
- Row 8 covers the conflict where both sides edited the file, and row 7 with `reset` set. The host wins.
- A change to only the executable flag on the host fails rows 4 and 7, so it becomes a put. The agent sees matching content in step 4.2 and only changes the file's permissions. A flag change counts as a host change, so it replaces a kept VM edit, like any other host change.
- Duplicate paths are ignored. `changes` is sorted, deletes first, so a case-only rename on a case-insensitive VM frees the name before the put. `kept` is sorted.

`AllPaths(local, remote)` gives the path set for a full comparison.

#### Copy instead of resending

Renaming a folder of 5,000 files produces 5,000 deletes and 5,000 puts, but the VM already holds every one of those files under the old names. A branch switch that moves files, and duplicate files such as a `LICENSE` in every package, have the same shape. Sending the content again costs a host read, an antivirus scan on a Windows host, and the transfer, for bytes the VM already has.

`PlanCopies(changes []Change, local map[RelPath]Entry, remote map[RelPath]RemoteEntry, noCopy set of RelPath) []Op` is pure. It turns `Diff`'s changes into the operations to send. An `Op` is a put, a copy with its `from` path, or a delete. For each put, look for a *source*: a path in `remote` whose `Hash` equals the put's `local.Hash`, and whose `Hash` equals its `Synced`, so the content on disk is known to be the host's. If one is found, send a copy from it. Otherwise, send the put. Also send a put, not a copy, when any of these is true:

- The put's path is in `noCopy`. The engine adds a path there for the rest of the session after a copy to it fails, so the next attempt sends the content.
- The source and the target differ only in letter case. On a case-insensitive VM, they are the same file, and the delete that follows would remove it.
- The target path, or one of its parent folders, is also the path of a delete in the same changes. That is a file turning into a folder, or the reverse, and the delete must happen first.

When several sources qualify, use the lowest path in sort order, so the plan doesn't depend on map order. Order the result as copies first, then deletes, then puts. Copies go first because their sources may be deleted in the same batch. Deletes still go before puts, so a case-only rename on a case-insensitive VM frees the name first.

The agent checks the source's hash before copying (agent step 5). If someone edited the source on the VM after the manifest, the copy fails instead of spreading the edit, and the path goes into `noCopy`.

`Pending` collects the paths one VM still has to check. It holds `{all bool, paths set of RelPath}` behind a mutex, and a channel with capacity 1 that signals "not empty". `Add(paths)` adds paths, or sets `all` when `paths` is `nil`. `Take()` returns the contents and clears them.

`Sync(ctx, Target{Name, Root}, repoID, reset, tree, pending, auth, out) error` runs one VM until `ctx` is cancelled, the agent sends `Fatal`, or `bootstrap.Dial` returns an error that retrying can't fix. It dials, runs a session, and reconnects after a disconnect. `out` prints each line prefixed with `[<Name>] `. See "One session".

### agent

`Serve(ctx, conn)` runs one session and returns when stdin closes, so the agent never outlives its SSH session. Windows OpenSSH does not reliably kill child processes, so exiting on end of input is required, not optional.

In the steps below, the agent's *current hash* of a path means this. If the path has a state record and the file's size and modified time match it, the current hash is the record's `hash`. If the path has a record that doesn't match, the agent hashes the file again and updates `size`, `modTime`, and `hash` in the record. If the path has no record, the agent hashes the file and creates no record. The agent applies the 2-second rule from the files section to every file it hashes or writes: a modified time with a zero sub-second part, less than 2 seconds before the moment of hashing, is stored as zero, so the next lookup hashes the file again.

0. Remove old builds of itself. Find its own folder with `os.Executable`, and set that folder's modified time to now with `os.Chtimes`. Delete every other folder in the parent `.sync-vm/agent/` folder whose modified time is more than 7 days old, with `os.RemoveAll`. Ignore every error, and do it in a goroutine so the session doesn't wait. The 7-day limit matters when two hosts with different `sync-vm` versions use the same VM account. Without it, each start would delete the other host's build, and every start would install again. On Windows, a build that another session is still running can't be deleted. It is removed by a later start.
1. Receive `Hello`. Check `Hello.Root` as described in "Check the destination" below. If a check fails, send `Fatal` with the reason and exit. Then open the resolved path with `os.OpenRoot` (Go 1.24+; use 1.25 for `Root.Rename`, `Root.MkdirAll`, and `Root.RemoveAll`). It confines every file operation to the root, symlinks included.
2. Take an exclusive OS lock (`flock` / `LockFileEx`) on `<root>/.sync-vm/lock`. If it's held, send `Fatal`. The OS releases the lock when the process dies, so a stale lock can't block anyone. If `Hello.Init` is true, apply the `init` result from check 5, then send `Manifest` with the resolved root and no files, and return. `init` sets up the folder but syncs nothing. Otherwise, delete everything in `<root>/.sync-vm/tmp/`. Anything there was left by a session that crashed.
3. Load `state.json`. Drop records whose file is gone. Compute the current hash of every remaining record, hashing in parallel as described under files. Send the records as `Manifest` messages of at most 10,000 files each, with the resolved root, and with `synced`, `hostSize`, `hostModTime`, and `exec` unchanged.
4. For each `Put`:
	1. If no file exists at the path, go to step 4.4.
	2. If the file's current hash equals `Put.hash`, read and discard the data frames. Create or update the path's record with the file's `size`, `modTime`, and `hash`. Set `synced` to `Put.hash`, `hostSize` to `Put.size`, `hostModTime` to `Put.modTime`, and `exec` to `Put.exec`. Apply the permissions rule below. Stop here. No content is written and nothing is reported.
	3. If the path has no state record, or its current hash differs from the record's `synced`, the file holds content the agent did not write. Copy it to `<root>/.sync-vm/overwritten/<path>`, replacing any earlier copy, and add the path to the next `Ack`'s `overwrote`.
	4. Write the data frames to a temporary file, verify the hash while writing, then rename the temporary file over the target. On Windows, retry briefly if the target is in use. Set the record's `size`, `modTime`, `hash`, and `synced` from the written file. Set `hostSize` to `Put.size`, `hostModTime` to `Put.modTime`, and `exec` to `Put.exec`. Add the path to the *unflushed list* used by "Save state durably". If the hash doesn't match, delete the temporary file and report the put as failed.
5. For each `Copy`: if the record for `Copy.from` is missing, or the current hash of `from` differs from `Copy.hash`, report the copy as failed with reason `source changed`. Otherwise, follow step 4 for `Copy.path`, with the content read from `from` instead of from data frames. Step 4.2 applies when the target already holds the content.
6. For each `Delete`: act only if the path has a state record. If its current hash differs from the record's `synced`, copy the file to `<root>/.sync-vm/overwritten/<path>` and add the path to `overwrote`. Remove the file. Then remove each parent folder that is left empty, up to the root, and ignore any error while removing a folder. On Windows, a folder can't be removed while a program on the VM uses it as its working folder, and that must not fail the delete. Drop the record.
7. Send `Ack` and save state as described in "Reply and save on a schedule".

**Temporary files.** Create each temporary file as `<root>/.sync-vm/tmp/<seq>.tmp`, not beside the target. Then a dev server watching the VM folder, such as the one `yarn start` or `dotnet watch` runs, sees only the final rename. A crash also can't leave temporary files in source folders, where a build might pick them up. Create the file with the permissions from the rule below, so the target never appears with the wrong ones. If the rename fails because the target is on a different filesystem, which happens when a folder inside the root is a mount point, delete the temporary file and write a new one as `.<name>.<seq>.tmp` beside the target instead.

#### Reply and save on a schedule

A reader goroutine decodes messages into a queue. The main loop takes operations from the queue and applies them in order.

- **Ack.** Send an `Ack` covering every operation applied so far when any of these happens: 32 operations since the last `Ack`, 50 ms since the last `Ack` with at least one operation applied, or the queue becomes empty. The host keeps up to 64 operations in flight, so replying every 32 lets it send more before it runs out. The agent can't otherwise tell that "no more input is waiting", because a read from a pipe just blocks.
- **Save.** Save state as described below at most once per second, and also when the queue has been empty for 200 ms, and when input ends. Saving only after replies would rewrite `state.json` about 150 times during a first sync of 10,000 files, at about 2 MB each. Saving later is safe: if the agent dies before a save, the files it wrote are recovered by row 5 or row 3, as the failure table describes.

#### Save state durably

A power loss can keep `state.json` but lose the content of files written just before it. A file that is then empty or cut short no longer matches its `synced` hash. That makes it look like a VM edit, and row 7 would keep it. To prevent that, the files must reach the disk before the record that describes them. Each save does these steps in order:

1. **Flush the files.** On Linux, call `unix.Syncfs` on the root's file descriptor. That one call flushes the whole filesystem. On macOS and Windows, open each path in the unflushed list, call `File.Sync`, and close it. Paths that are gone are skipped. Then clear the list.
2. **Write the state.** Write `state.json.tmp` in `<root>/.sync-vm/`, call `File.Sync` on it, and close it.
3. **Replace the state.** Rename `state.json.tmp` over `state.json`. On Linux and macOS, also open `<root>/.sync-vm/` and call `File.Sync` on it, so the rename itself reaches the disk.

If `state.json` can't be parsed when the agent loads it, it is damaged. The agent sends `Fatal` with rejection D from check 5, and doesn't guess at its content.

**Permissions rule.** On Linux and macOS, give a file mode `0755` when `exec` is true and `0644` when it is false. The process umask still applies when a file is created, as it does for any program. When step 4.2 changes only the flag, use `os.Chmod`. On Windows, ignore `exec`, but still record it, so the host doesn't send the same put again.

The agent never lists or deletes files it didn't write. It does replace a file it didn't write when a `Put` targets that path, after saving a copy. That makes it safe to point `init` at a shared folder with `-force`.

#### Check the destination

`os.OpenRoot` keeps every write inside the destination, but it doesn't check the destination itself. A mistyped path, such as `C:\Windows\System32` or the home folder, would otherwise get the repo's files written into it. Windows OpenSSH users are often administrators, so the OS doesn't reliably refuse. The agent runs these checks in order, before it creates or opens anything. Each failure sends `Fatal` with a message that names the path and the rule it broke.

1. **Expand `~`.** If `Hello.Root` is `~`, or starts with `~/` or `~\`, replace the `~` with `os.UserHomeDir()`. No other form of `~` is expanded.
2. **Absolute path.** Reject the path unless `filepath.IsAbs` is true on the VM's OS. A relative path would resolve against the SSH login folder, which the user may not expect.
3. **Resolve links.** If the path exists, replace it with the result of `filepath.EvalSymlinks`. If it doesn't exist, resolve its nearest existing parent folder and append the remaining segments. Run the later checks, and every later file operation, on the resolved path. A link that points into a system folder is then caught. Reject the path if it exists and is not a directory.
4. **Not a system folder.** Reject the resolved path if it matches the list below. Compare paths after `filepath.Clean`, ignoring letter case on Windows and macOS. Read each environment variable on the VM. Skip an entry whose variable is empty.

	| OS | Reject the path itself | Reject the path and everything inside it |
	|---|---|---|
	| Linux | `/`, `/home`, the user's home folder (`os.UserHomeDir`) | `/bin`, `/boot`, `/dev`, `/etc`, `/lib`, `/lib32`, `/lib64`, `/proc`, `/root`, `/run`, `/sbin`, `/sys`, `/usr`, `/var`, `~/.sync-vm` |
	| macOS | `/`, `/Users`, the user's home folder | `/Applications`, `/bin`, `/dev`, `/etc`, `/Library`, `/private`, `/sbin`, `/System`, `/usr`, `/var`, `~/.sync-vm` |
	| Windows | every drive root (`C:\`, `D:\`, and so on), `%SystemDrive%\Users`, the user's home folder (`%USERPROFILE%`) | `%SystemRoot%`, `%ProgramFiles%`, `%ProgramFiles(x86)%`, `%ProgramData%`, `%USERPROFILE%\AppData`, `%USERPROFILE%\.sync-vm` |

	The list catches common mistakes. It is not a complete list of dangerous folders. Check 5 is the main protection.
5. **Set up by this repo's `init`.** A folder is a *destination* when it contains `.sync-vm/state.json`. Its *owner* is the `repo` in that file. Find the folder's state in the first column, then apply the column for the command.

	| Folder | `sync-vm` | `sync-vm init` | `sync-vm init -force` |
	|---|---|---|---|
	| Doesn't exist | Reject A | Create it, write `state.json` with this repo and no files | Same as `init` |
	| Empty | Reject A | Write `state.json` with this repo and no files | Same as `init` |
	| Has files, not a destination | Reject A | Reject C | Write `state.json` with this repo and no files |
	| Destination owned by this repo | Accept | Accept, change nothing | Accept, change nothing |
	| Destination owned by another repo | Reject B | Reject B | Set `repo` to this repo, keep every record |
	| `state.json` can't be parsed | Reject D | Reject D | Replace `state.json` with this repo and no files |

	Rejection messages:
	- **A.** `<path> is not a sync-vm destination. Run sync-vm init <target> to set it up.`
	- **B.** `<path> is synced from a different repo. Pick another folder with sync-vm init <target>:<path>, or rerun init with -force to take it over.`
	- **C.** `<path> already has files. Check the path. To set it up anyway, rerun init with -force.`
	- **D.** `<path>/.sync-vm/state.json is damaged. Run sync-vm init <target> -force to rebuild it. The host then sends every file again.`

	The host fills in `<target>`, because the agent doesn't know it.

	A normal run never creates a folder or changes its owner. A system folder or a home folder is never empty, so `init` without `-force` also refuses them, even if the list in check 4 misses them. Taking over a destination keeps its records, because they still describe files on disk correctly. The host then sends whatever differs, as usual.

`-force` changes only check 5. Checks 1 to 4 always run, for every command.

## Syncing several VMs

`sync-vm` with several selected targets runs them in one process:

1. Create one `Tree` for the repo, and one `bootstrap.Auth`. Every VM shares the `Auth`, so a key unlocked for one VM is unlocked for all of them.
2. Create one `Pending` per target.
3. Start `files.Watch` once. Add each batch to every target's `Pending`.
4. Start one `engine.Sync` per target in its own goroutine.
5. On Ctrl+C, cancel the context and wait for every `Sync` to return.
6. Exit with status 0 if no `Sync` returned an error, and nonzero otherwise.

The VMs share the file listing, the hash cache, and the watcher, so the host reads each changed file once, not once per VM. Everything else is per VM. A disconnect, a `Fatal`, or a slow VM affects only that VM. The others keep syncing.

## One session

`Sync` repeats these steps for one VM.

1. `Take` from this VM's `Pending` and discard the result. The full listing in step 3 covers those paths. Events that arrive after this point stay in `Pending` for step 4.
2. `bootstrap.Dial` with the shared `Auth`. If it returns an error that retrying can't fix, such as a failed login or a changed host key, go to step 7. Any other error counts as a disconnect (step 6). Send `Hello` with the saved root and the repo ID, and receive `Manifest` messages until one has `more` false. On `Fatal`, go to step 7. The manifest becomes the host's picture of the VM. Pass it to `Tree.Seed`. On the first successful connect in this process, if the platform's OS is `linux` or `darwin`, run the line-ending check below.
3. Run `Diff(AllPaths(local, remote), local, remote, reset)`, where `local` comes from `Tree.List`. `Tree.List` runs after `Tree.Seed`, so on a fresh start it reads only files that changed since the last run. Print one line per path in `kept`: `kept VM edit: <path>`. Turn the changes into operations with `PlanCopies`, and send them without waiting for each ACK, with up to about 64 operations in flight. That window size is a starting guess to tune.
4. Loop: wait for this VM's `Pending` to signal, then `Take`. If `all` is set, use `Tree.List`. Otherwise, use `Tree.Stat` on the paths. Run `Diff` with `reset` false, plan with `PlanCopies`, and send the operations. Don't print `kept` here, because the same paths would repeat on every batch.
5. On each `Ack`, update the picture of the VM. For an acknowledged put or copy, set the entry to `{Size, Hash, Synced: Hash, HostSize: Size, HostModTime: ModTime, Exec}` from the operation. For an acknowledged delete, remove the entry. Remove each failed path from the picture. It is retried with the next batch, with growing delays. If a failed operation was a copy, also add its path to `noCopy`, so the retry sends the content. For each path in `overwrote`, print `replaced VM edit: <path>. The VM's version is in <root>/.sync-vm/overwritten/<path>. Rerun any VM commands that changed it.`
6. On disconnect, print `disconnected, retrying`, wait, and return to step 1. Start the wait at 1 second and double it after each failed attempt, up to 30 seconds. Reset it after a session reaches step 4. There is no resume logic. The manifest exchange makes a fresh start correct. `-reset` applies only to the first session's step 3. A reconnect runs `Diff` with `reset` false.
7. On `Fatal`, or a `Dial` error that retrying can't fix, print the reason and return an error. Don't reconnect, because the same check would fail again.

During a session, the host does not see VM edits as they happen, because the agent does not watch the VM folder. A VM edit made mid-session stays until the host changes that file. The agent then detects the edit in step 4.3 and reports it. If the edited file is the source of a copy, the copy fails in step 5 and the host sends a put instead, so the edit doesn't spread to another path. At the next session start, the manifest shows it and row 7 keeps it.

### Guards before sending

Steps 3 and 4 apply two guards before running `Diff`:

- **A git error is not an empty repo.** If `Tree.List` or `Tree.Stat` returns an error, print `git failed: <error>. Retrying in 2 seconds.` Send nothing. After 2 seconds, `Add(nil)` to this VM's `Pending`, so the next pass is a full `List`. Keep the session open.
- **Never delete everything.** If `local` is empty and the picture of the VM has at least one file, print `The host has no files to sync, but <target> has <n>. Not deleting anything. Check the repo, then restart sync-vm.` Send nothing, and stop processing batches for this VM until `sync-vm` restarts. A repo that is really empty has nothing worth syncing, so this costs nothing in normal use, and it stops a bad listing from wiping the VM.

### Tell the user when a VM is up to date

The user edits on the host, then switches to the VM to build. They need to know when the VM has the edit. Keep two counters per session, files put and files deleted, since the last report. When the number of unacknowledged operations reaches zero and this VM's `Pending` is empty, report and reset the counters:

- After step 3, always print `up to date (<n> files)`, where `<n>` is the number of files in the picture of the VM.
- After that, print `synced: <p> updated, <d> deleted` only when either counter is nonzero. Omit a part whose counter is zero.

Count a put that only changed the executable flag as updated. Each line gets the `[<target>] ` prefix, like all output from `Sync`.

### Check line endings

On a Windows host with git's `core.autocrlf` set to `true`, files in the working tree have Windows line endings (CRLF). A shell script with CRLF fails on Linux and macOS with errors like `/bin/sh^M: bad interpreter`. `sync-vm` doesn't convert line endings, because the hash must match the bytes on disk. It warns instead.

Call `Tree.CRLFScripts` once per process, and reuse the result for every VM. If it returns any files, print once per Linux or macOS VM:

```
[linux] 3 scripts have Windows line endings and will fail on this VM: gradlew, scripts/build.sh, mvnw
	To fix them, add these lines to .gitattributes and commit:
		*.sh text eol=lf
		gradlew text eol=lf
		mvnw text eol=lf
	Then delete those files and run git checkout -- <files> to write them again with LF line endings.
```

List at most 5 files in the first line and add `and <n> more` after them. Don't stop syncing. The VM may still be usable for work that doesn't run those scripts.

## Failure behavior

| Situation | Result |
|---|---|
| Agent dies after a write, before saving state | The next manifest shows the file with `hash` different from `synced`, or the file has no record. Either way, the host sends a put (row 5 or row 3). The agent finds the content already matches, records it as synced, and reports nothing. |
| Agent dies after a delete, before saving state | The stale record is dropped on load because the file is gone. |
| A VM file is locked by another program | The rename is retried briefly, then reported as failed. Other files keep flowing. The host retries later. |
| A host file changes while being sent | The agent's hash check fails, and the host retries. |
| Someone edits a synced file on the VM, and the host's copy is unchanged | The edit stays. Each session start prints `kept VM edit: <path>`. |
| Someone edits a synced file on the VM, then the host's copy changes | The host's version replaces it. The VM's version is saved under `.sync-vm/overwritten/`, and the host prints an alert. |
| One of several VMs is off or unreachable | That VM retries with growing waits. The other VMs keep syncing. |
| A VM folder is wiped, or the VM is rebuilt | That VM stops with rejection A. This also happens on a reconnect, if the folder was wiped while `sync-vm` ran. The other VMs keep syncing. After `sync-vm init <target>`, the manifest is empty, so the host sends every file. |
| `state.json` is deleted, but synced files remain | The folder is no longer a destination. `init` refuses it with rejection C. The user runs `sync-vm init <target> -force`. The manifest is then empty, so the host sends every file. Files whose content differs from the host's are saved under `.sync-vm/overwritten/` and reported. The host wins. |
| The path given to `init` is wrong | If it is relative, is a system folder, or points to one through a link, `init` refuses it. If it already has files, `init` refuses it unless the user passes `-force`. Nothing is written or saved in either case. |
| Two host repos, such as two clones or two worktrees, point at the same VM folder | The second repo's `init` gets rejection B, because the folder's owner is the first repo. Neither repo's daily run touches a folder owned by the other. |
| The host repo is deleted and cloned again | The new clone has no `sync-vm.json`, so `init` creates a new repo ID. `init` on the old VM folder gets rejection B. `init -force` takes it over, keeping its records. |
| The host repo folder is moved or renamed | `sync-vm.json` is in the git folder and moves with it. Nothing changes. |
| `git ls-files` fails, for example while another git command holds `.git/index.lock` | `sync-vm` prints the error, sends nothing, and retries a full listing every 2 seconds until it succeeds. |
| The host listing is empty but the VM has files | `sync-vm` deletes nothing on that VM, prints a warning, and stops syncing to that VM until it restarts. |
| A file is created under an ignored folder on the host, such as `bin/` | `Tree.Stat` filters it out with git's ignore rules, so it doesn't sync. |
| `.gitignore` changes | The watcher triggers a full listing, so newly ignored files are deleted from each VM and newly unignored files are sent. |
| A script is marked executable in git, or loses the mark | The host sends a put. The agent changes only the file's permissions. |
| The SSH key has a passphrase | `sync-vm` asks once, when the first VM that uses the key connects. Every other VM and every reconnect reuses the unlocked key until `sync-vm` exits. |
| The passphrase is typed wrong | `sync-vm` asks again, up to 3 tries. After that, it skips the key and tries the next login method. |
| A VM accepts only passwords | `sync-vm` asks for that VM's password once per run, and reuses it for reconnects. If a reconnect is rejected, it forgets the password and asks again. |
| A VM is connected to for the first time | `sync-vm` shows the host key fingerprint and asks whether to trust it. On yes, it adds the key to `~/.ssh/known_hosts`. On no, that VM stops. |
| A VM's host key has changed | That VM stops with a warning and doesn't retry. If the VM was rebuilt, the user removes its old line from `~/.ssh/known_hosts`. |
| `sync-vm` runs without a terminal and needs to ask something | That VM stops with a message saying it needs a terminal. Nothing waits for input that can't arrive. |
| Two VMs need a prompt at the same time | The prompts take turns. Each one names its VM. |
| A VM sleeps or loses its network without closing the connection | The keepalive gets no reply for about 45 seconds and closes the connection. That VM reconnects without prompting. |
| A host repo uses CRLF line endings and syncs to a Linux or macOS VM | Each such VM prints a warning naming the scripts that will fail, and how to fix them. Syncing continues. |
| A host file's content changes while its size and modified time stay the same, for example a copy tool that keeps the old date | The host doesn't notice, and the VM keeps the old content until the file changes again. Git's `git status` has the same blind spot. Changing the file's modified time on the host, for example with `touch`, fixes it. |
| The user runs `sync-vm -reset` | Every kept VM edit on every selected VM is replaced, saved, and reported. |
| Two `sync-vm` processes sync the same repo to the same VM | The second one gets `Fatal` from the lock for that VM. Its other VMs, if any, sync normally. |
| An older agent is running during an upgrade | The new build installs to a different folder. Nothing is overwritten, which Windows would refuse anyway. |
| Two hosts with different `sync-vm` versions use the same VM account | Each agent removes only builds unused for 7 days, so neither deletes the other's build. |
| The VM loses power right after the agent wrote files | Each save flushes the written files before it writes `state.json`, so `state.json` never describes content that didn't reach the disk. A file lost in the power cut either has no record or still has its old `synced` hash. Either way the host sends it again, and nothing broken is kept as a VM edit. |
| `state.json` is damaged | Every command stops with rejection D. `sync-vm init <target> -force` replaces it, and the host then sends every file. Files on disk that differ are saved under `.sync-vm/overwritten/` first. |
| The agent dies while writing a file | The temporary file is in `<root>/.sync-vm/tmp/`, not in a source folder. The next session deletes it. |
| A dev server on the VM watches the synced folder | It sees only finished files appear through a rename, never temporary files. |
| A host file changes more often than every 150 ms | Batches still go out at least once a second, so other changes keep syncing. |
| `git rm --cached` or `git add -f` changes which files sync | The change to `<git-dir>/index` triggers a full listing within about 2 seconds. |
| An editor saves through a temporary file beside the real one | Untracked names on the editor temporary-file list never sync, so the VM sees only the real file change. |
| A folder renamed on the host holds files the VM already has | The host sends copies, and the agent copies the files locally. No content crosses the connection. |
| The source of a copy was edited on the VM | The copy fails with `source changed`, and the host sends the content as a put instead. |
| A folder left empty by a delete is in use on a Windows VM | The file is deleted, and the folder stays. The delete still succeeds. |
| The VM holds more than about 250,000 files | The manifest arrives in several messages of up to 10,000 files, so no frame passes the 64 MB cap. |

A missing or damaged state record never keeps a VM edit. With no record, the path is absent from the manifest, so row 3 sends the host's version. Only a record whose `synced` matches the host's current content can keep a VM edit.

## Removed from the old design

- TCP over SSH port tunnels, and the agent's daemon mode
- Running `ssh.exe` and `scp`. The built-in SSH client in bootstrap replaces both.
- The host-side hash cache on disk, which was a second, staler copy of state the agent owns. The in-memory cache under files replaces it. Between runs, it is seeded from the agents' records, so the agents' records are still the only file state on disk.
- The `gitcheck` branch-behind warning. Git is now used for the file list and to find the repo.
- `RENAME`, `MKDIR`, `RMDIR`, `SNAPSHOT_START`, `SNAPSHOT_END`
- Sequence-number resume after reconnect
- The `-w` and `-dest` flags. `init` takes the target and path once, and `sync-vm.json` remembers them.
- Syncing a folder that isn't a git repo, by walking the tree

## Alternatives rejected

- **Replaying watcher events, as the old design did.** The VM drifts whenever the host's memory is wrong, and the first sync and watch mode stay two code paths.
- **The host keeping a VM's file list on disk.** That list goes stale whenever the VM changes on its own.
- **No agent, with `tar` over SSH.** Every batch pays for a new SSH process. There's no cheap way to list the VM's files and no atomic writes.
- **Always replacing VM edits at session start.** That is simpler, and the VM always matches the host. But every `sync-vm` start or reconnect reverts `yalc add` and `yarn install` edits, so the user reruns them after each one.
- **Two-way sync, copying VM edits back to the host.** It needs a watcher in the agent, writes into the host working copy, and rules for new and deleted VM files. It also copies VM-only edits, like the `.yalc/` path that `yalc add` writes into `package.json`, into the host repo, where they are easy to commit by mistake.
- **Saving the VM list in the repo's working tree, such as `.sync-vm.json` next to `package.json`.** It is easy to commit and push by mistake.
- **Saving the VM list in the user's config folder, keyed by repo path.** Moving or renaming the repo loses the link. Entries for deleted repos pile up.
- **One `sync-vm` process per VM.** Each process would list, hash, and watch the same repo separately.
- **Running `ssh.exe` and letting it prompt.** Each `ssh` process asks for the passphrase again. That means several prompts per connection, and one more set on every reconnect, often while the user is away. Windows OpenSSH has no `ControlMaster` to share one connection.
- **Feeding the passphrase to `ssh.exe` through `SSH_ASKPASS`.** It needs OpenSSH 8.4 or newer for `SSH_ASKPASS_REQUIRE`, and many Windows 10 machines ship 8.1. Passing the secret safely to each child process is also fiddly.
- **A private ssh-agent that only `sync-vm`'s `ssh` processes use.** It works, but it is an ssh-agent under another name. Where ssh-agent is blocked by policy, it likely breaks the same rule.
- **`github.com/skeema/knownhosts`, `github.com/kevinburke/ssh_config`, and `github.com/pkg/sftp`.** They do what the in-house code does, but company policy allows only the standard library, `golang.org/x/...`, and `fsnotify`. Vendoring them with `go mod vendor` doesn't help, because vendored code is still third-party code.
- **Installing the agent with shell commands instead of SFTP**, such as `cat > file` on Linux, or base64 through `certutil` or PowerShell on Windows. Each shell needs its own command, PowerShell's input isn't safe for binary data, and cmd's command length limit makes a base64 upload take over a thousand round trips. Running `scp -t` on the VM needs `scp` there and a rename command per shell. SFTP avoids the shell entirely.
- **Requiring a key without a passphrase.** That is the user's choice to make, not `sync-vm`'s. It still works: such a key never prompts.

Mutagen installs its agent over SSH in a similar, versioned way. It's worth reading as prior art, though the requirements rule out using it.

## Build order

Each step ends in a check before the next starts.

1. Write `protocol` and `Diff`, and the dependency test: it reads `go.mod` and fails if any `require` line names a module other than `golang.org/x/...` or `github.com/fsnotify/fsnotify`. Unit tests with plain data, including a round trip of every message, a put larger than one data frame, a traversal path rejected on decode, a `.sync-vm/...` path rejected on decode, and one case per `Diff` row, with and without `reset`. Add cases where only `Exec` differs, for rows 4 and 7. Round-trip a `Copy`, and a manifest of 25,000 files, which must arrive as three messages. Test `PlanCopies` with plain data: a folder rename becomes copies then deletes, a duplicate file becomes a copy, a VM-edited source is not used, a path in `noCopy` stays a put, a case-only rename stays a put, a file that turns into a folder stays a put, and with several sources the lowest path is chosen.
2. Write `agent`. Test in-process against temp folders over an in-memory pipe. Cover each branch of step 4: a new file, matching content, an unedited file, an edited file (check `overwrote` and the saved copy), and a file with no state record. Cover a delete of an edited file, and a crash simulated by writing a file without saving state. Cover each destination check: `~` expansion, a relative path, a path in the system-folder list, and a link that points into one, each with every command. Then a file instead of a folder. Then every cell of the check 5 table. Check that a normal run never creates a folder or changes an owner, and that a takeover keeps every record. Build the system-folder list from injected environment values, so the Windows rows run in tests on any OS. On Linux and macOS, check that a put with `exec` true creates mode `0755`, and that a put with only the flag changed calls `os.Chmod` without writing content. Check that step 0 removes sibling folders under `.sync-vm/agent/` older than 7 days, and keeps newer ones and its own. Check that temporary files are created under `.sync-vm/tmp/` and never beside the target, and that a leftover file there is deleted at the next session start. Check a copy with a matching source, a missing source, and a source edited on the VM. Check a delete whose parent folder can't be removed: the delete still succeeds. Check the `Ack` schedule with a fake clock: after 32 operations, after 50 ms, and when the queue empties. Check that 10,000 puts in a row save `state.json` at most once per simulated second. Check the save order with a fake filesystem that records calls: flush, write `state.json.tmp`, sync it, rename, sync the folder. Simulate a power loss by keeping `state.json` but truncating a file written after the last flush, and check that the host sends it again instead of keeping it. Check that a damaged `state.json` gives rejection D, and that `init -force` replaces it.
3. Write `repo` and `files`. Test against temp git repos. For `repo`, cover `ParseTarget` with a Linux path, a Windows path, and no path. Cover a `Load` and `Save` round trip, and `Find` in a worktree, which must return the worktree's own git folder. For `files`, check that `List` doesn't open a file whose size and modified time match a cache record, using a counting file opener. Check the same for a record added by `Seed`. Check the 2-second rule with an injected clock and file times: a time with a zero sub-second part, less than 2 seconds old, gets `ModTime` zero and is hashed again on the next `List`. A time with a nonzero sub-second part keeps its value. Check that two `List` calls at once run git and hash each file only once. Check that `Stat` on a known path runs no git command, by counting git calls. Check that each name on the editor temporary-file list is skipped when untracked and kept when tracked. Check that `Watch` sends a batch within 1 second while a file changes every 100 ms, and sends one `nil` after several index changes within 2 seconds. Run `List` from several goroutines with the race detector on. Check that `List` sets `Exec` from mode `100755`, skips a submodule and a symlink with a warning, and includes an ignored `.env` only when an allowlist pattern matches it. Check that `List` returns an error, not an empty map, while `.git/index.lock` exists and git refuses to run. Check that `Stat` drops a new file under an ignored folder, keeps a new untracked file that isn't ignored, and falls back to `List` above 500 paths. Check that `Watch` drops events under `.git/` and sends `nil` for a `.gitignore` change. Check `CRLFScripts` against a repo with `core.autocrlf=true`, containing a `.sh` file, an executable file, and a plain text file.
4. Write `engine` and `cmd/sync-vm`. Test end to end in one process, with no SSH: create, edit, delete, rename, and kill the connection mid-sync, and check that the folders converge. Then edit a file on the VM side, reconnect, and check that the edit is kept. Change that file on the host and check that it is replaced and reported. Run once with `reset` and check that every VM edit is replaced. Restart the host with a new `Tree`, change one host file, and check that the host opens only that file. Sync to two in-process agents at once, kill one connection, and check that the other keeps converging. Check that each changed host file is opened once, not once per VM. Rename a folder of 1,000 files on the host, and check that no host file is opened and that every operation sent is a copy or a delete. Edit the source of a planned copy on the VM side first, and check that the failed copy is retried as a put. Run `init` twice for one target with different paths and check the second is refused. Check that `init` refuses the target names `init`, `vm`, and `allowlist`. Check `vm remove`, and check `vm list` output with no VMs, one VM, and two VMs. Check `allowlist add` with a new pattern, a duplicate, and a pattern git rejects. Check `allowlist remove` with a saved pattern and an unsaved one, and check `allowlist list`. Make the host listing fail and check that nothing is sent and the next pass retries. Make the host listing empty while the VM has files, and check that nothing is deleted. Check the output lines: `up to date (<n> files)` once after the initial sync, `synced: ...` only after batches that changed something, and nothing for a batch with no changes. Test argument parsing as a table: flags before, after, and between words; `--` before a word that starts with `-`; `--force` and `-force` alike; `-reset` with `init` rejected with status 2; the wrong number of words for each command; `vm` and `allowlist` with a missing or unknown subcommand; `-h` and `-version` exiting 0 outside a git repo; and `-h` winning over every other argument.
5. Write `bootstrap` and `scripts/build-dist.sh`, which builds every agent into `internal/bootstrap/agents/` before building `sync-vm`. Test `internal/sshconfig` with table tests: `Key value` and `Key=value`, quoted values, first value wins across blocks, `IdentityFile` collecting from several blocks, `*`, `?`, and `!` patterns, a `Match` block skipped, `Include` with a relative path, a glob, and inside a `Host` block, a missing included file skipped, a self-include stopped at 16 levels, each `%` token, an unknown token skipped, and a bad `Port`. Test the known-hosts algorithm lookup against `known_hosts` files with a plain entry, a hashed entry, a `[host]:2222` entry, a wildcard entry, and an RSA entry, which must give the three RSA names in order. Test `internal/sftp` against OpenSSH's real `sftp-server` binary, run as the `sftp` subsystem of an in-process SSH server built with `golang.org/x/crypto/ssh`. Look for it at `/usr/lib/openssh/sftp-server`, `/usr/libexec/sftp-server`, `/usr/libexec/openssh/sftp-server`, `C:\Windows\System32\OpenSSH\sftp-server.exe` (installed with the Windows OpenSSH server feature, not the client), and `C:\Program Files\Git\usr\lib\ssh\sftp-server.exe` (shipped with Git for Windows), and skip the test with a message when none exists. Cover `Stat` of a missing path, `MkdirAll` of a nested path, a `WriteFile` of 1 MB that reads back identical, `Chmod`, and both renames. Then test `bootstrap` in-process against an in-process SSH server with that `sftp` subsystem, with prompts answered by a fake prompter that counts calls:
	- Two dials with the same passphrase-protected key, then a reconnect, prompt exactly once.
	- A wrong passphrase is asked again, up to 3 tries.
	- A password-only server and a keyboard-interactive server each prompt once per run. A rejected stored password is forgotten and asked again.
	- An unknown host key prompts, and on yes adds a line to a temporary `known_hosts`. On no, the dial fails without retry.
	- A changed host key fails without a prompt and without retry.
	- With `known_hosts` holding only an Ed25519 key, a server that also offers RSA still connects.
	- Without a terminal, a dial that needs a prompt fails with the terminal message.
	- `~/.ssh/config` handling: `HostName`, `User`, `Port`, repeated `IdentityFile`, `ProxyJump`, `Include`, and `user@` in the target winning over `User`.
	- A server that stops answering is closed by the keepalive within 60 seconds.
	- An install writes the agent through SFTP, and a second dial finds it and doesn't upload it again.

	Then test against a real Linux VM and a real Windows VM, with both cmd and PowerShell as the login shell, and with both VMs synced from one `sync-vm` process using one passphrase-protected key. Check that the passphrase is asked once. Check that `./gradlew` from a Windows host runs on the Linux VM. Pause a VM mid-session and check that its sync reconnects after it resumes, without a prompt. This is the only step that needs real VMs.

## Open questions

1. **Symlinks and submodules on the host.** Both are skipped with a warning. Following a symlink could pull in large trees. A submodule is its own repo and would need its own file listing. Supporting either is deferred until a repo needs it.
2. **Empty folders.** Git doesn't track them, so they won't sync. This is assumed acceptable.
3. **Repeated `kept VM edit` lines.** A long-lived VM edit, such as `yalc add` in `package.json`, prints on every session start. The user may learn to ignore the list. Suppressing repeats is deferred until the noise is a real problem.
4. **First sync into a folder that already has files.** This needs `sync-vm init -force`. Every existing file the host also has, with different content, is copied to `.sync-vm/overwritten/`. For a large folder, the copies use real disk space. Nothing cleans `.sync-vm/overwritten/` yet.
5. **Platforms.** The assumption is Linux, Windows, and macOS on amd64 and arm64, six agent builds. The ~30 MB size is an estimate, not a measurement.
6. **Permission changes made on the VM.** The agent sets permissions only when it writes a file, and doesn't notice if something on the VM changes them later. That is assumed acceptable.
7. **Applying allowlist changes while running.** A running `sync-vm` reads the allowlist only at start. Watching `sync-vm.json` for changes is deferred.
8. **SSH settings beyond the supported list.** `ProxyCommand`, `ProxyJump` with more than one hop, `Match` blocks, certificates, and hardware-backed keys aren't supported. Each can be added when a VM setup needs it.
 