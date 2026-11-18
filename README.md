# sync-dir

Syncs a local folder and a remote folder over SSH with rsync. It never deletes anything.

- **Direction:** `push` sends local → remote, `pull` sends remote → local.
- **Newer wins:** a file is copied only if the source copy is newer (by modification time). A newer file in the destination is kept.
- **Nothing is deleted:** files missing in the source, and the old versions of replaced files, are moved to `<destination>/.deleted/<timestamp>/`.
- **Dry run first:** every run starts with a dry run and prints a report of the changes. A real run asks for confirmation before it applies them.

## Requirements

- **GNU rsync 3.x on the local machine.** On macOS, `/usr/bin/rsync` is openrsync and the script refuses to use it. Install GNU rsync with `brew install rsync`. The script finds `/opt/homebrew/bin/rsync` or `/usr/local/bin/rsync` automatically, or you can set `RSYNC_BIN`.
- **rsync on the remote host.** If it isn't on the remote `PATH`, set `REMOTE_RSYNC`.
- **SSH login with a key or ssh-agent.** The script never stores passwords.
- **Correct clocks** on both machines, because "newer" is decided by modification time.

## Configuration

Edit the variables at the top of `sync-dir`, or put the same variables in a config file and pass it with `-c`:

| Variable | Default | Meaning |
| --- | --- | --- |
| `LOCAL_DIR` | `$HOME/sync/project` | Local folder |
| `REMOTE_HOST` | `user@server.example.com` | SSH target. Empty means `REMOTE_DIR` is a local path (useful for testing). |
| `REMOTE_DIR` | `/home/user/sync/project` | Folder on the remote host |
| `SSH_PORT` | `22` | SSH port |
| `SSH_KEY` | empty | Path to the private key. Empty uses the ssh defaults or the agent. |
| `REMOTE_RSYNC` | `rsync` | rsync binary on the remote host |
| `DIRECTION` | `push` | Default direction: `push` or `pull` |
| `MIRROR_DELETES` | `1` | `1` moves destination files that are missing in the source to `.deleted`. `0` leaves them in place. |
| `DELETED_DIR` | `.deleted` | Name of the backup folder inside the destination |
| `EXCLUDES` | `(".DS_Store" "*.swp")` | rsync exclude patterns. Excluded files are never touched in the destination. |

Example config file `~/work.conf`:

```bash
LOCAL_DIR="$HOME/projects/report"
REMOTE_HOST="me@build01.example.com"
REMOTE_DIR="/srv/report"
SSH_KEY="$HOME/.ssh/id_ed25519"
EXCLUDES=(".DS_Store" "*.swp" "node_modules/" ".git/")
```

## Usage

```text
sync-dir [push|pull] [-n] [-y] [-k] [-c config_file]
```

| Option | Meaning |
| --- | --- |
| `push` / `pull` | Direction. Overrides `DIRECTION`. |
| `-n` | Dry run only: print the report and change nothing |
| `-y` | Apply without asking for confirmation |
| `-k` | Keep extra destination files instead of moving them to `.deleted` |
| `-c FILE` | Load configuration from `FILE` |
| `-h` | Help |

Examples:

```bash
sync-dir push -n -c ~/work.conf   # preview local -> remote
sync-dir push -c ~/work.conf      # preview, confirm, apply
sync-dir pull -y -c ~/work.conf   # remote -> local, no question
sync-dir pull -k -c ~/work.conf   # pull, keep local files missing on the remote
```

## Report

```text
=== DRY RUN: push  /Users/me/projects/report/  ->  me@build01:/srv/report/ ===
New files (2):
  + new.txt
  + sub/same.txt
Updated files (old version -> .deleted/2026-11-18_200158) (1):
  ~ upd.txt
Moved to .deleted/2026-11-18_200158 (missing in source) (1):
  - gone.txt
New directories (1):
  d sub/
Total transferred file size: 12 bytes
Apply these changes? [y/N]
```

After a real run, the same report is printed again with the heading `APPLIED`.

## How it works

The script runs rsync with these options:

```text
-rlpt --update --backup --backup-dir=.deleted/<timestamp> --exclude=/.deleted/ [--delete] -e "ssh -p <port> [-i <key>]"
```

- `--update` skips files that are newer in the destination.
- `--delete` together with `--backup-dir` moves files into `.deleted/<timestamp>/` instead of deleting them. The same happens to the previous version of every overwritten file.
- `.deleted` is excluded, so it is never synced or cleaned up by the script.
- Owner and group are not copied (`-rlpt`), so the sync works between machines with different users.

## Limitations

- **Edits on both sides:** this is one-way sync. If a file changed on both sides, the newer copy wins. When the newer copy is in the source, the older destination copy is kept in `.deleted`. When it is in the destination, nothing happens. Run `push` and `pull` to bring both sides up to date.
- **Small time window:** files that change between the dry run and the real run can make the final result differ slightly from the preview.
- **`.deleted` keeps growing:** clean it up yourself, e.g. `find <dir>/.deleted -mindepth 1 -maxdepth 1 -mtime +30 -exec rm -rf {} +`.
- **No spaces in the key path:** `SSH_KEY` must not contain spaces, because rsync splits the `-e` string on spaces.
- **Exit code 24:** "some source files vanished during transfer" is treated as a warning. Any other rsync error stops the script.

## Setup

```bash
chmod +x ~/bin/sync-dir
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc   # then open a new terminal
```
