# rsnap-backup

`rsnap-backup` is a simple snapshot backup script for Linux written in bash and built on `rsync`.

It creates timestamped backup directories that are directly browseable with normal file tools. Each backup looks like a full copy, but unchanged files are hard-linked from the previous backup so they do not consume additional space.

This gives you a backup style similar to Back In Time, while keeping the implementation small, transparent, and under your control.

The home for this system is:  https://github.com/blakemcbride/Rsnap-backup

## Design and implementation credit

The design of this backup system is by **Blake McBride**.

The bash implementation in this repository was produced with **ChatGPT**, based on Blake McBride's design requirements and then refined through iterative review and testing.

## Features

- Plain bash script
- Uses `rsync`
- Timestamped snapshot directories
- Directly browseable backups
- Hard-link reuse of unchanged files between snapshots
- INI-style configuration file
- Multiple source paths
- Exclude patterns passed to `rsync`
- Retention policy support:
  - keep all backups from the last N days
  - keep the last backup of each older month
- Safe deletion of old snapshots with `rm -rf`
- Designed to be run as `root` when full-system access is required

## Why this exists

This script was created for users who want:

- a backup layout similar to Time Machine or Back In Time
- no dependency on a larger backup application
- no repository/archive format
- direct access to files in every snapshot without restore tools
- a transparent script they can inspect, modify, and maintain themselves

## How it works

Each backup is stored in a directory named like this:

```text
2026-03-22_13-41-45
```

Inside that directory, the original filesystem layout of each configured source is preserved.

For example, if the config file contains:

```ini
[sources]
path=/home/username/Music
path=/home/username/Documents
```

then a snapshot may contain:

```text
2026-03-22_13-41-45/home/username/Music
2026-03-22_13-41-45/home/username/Documents
```

On the first run, a full backup is made.

On later runs, the script finds the most recent prior snapshot and uses `rsync --link-dest` so unchanged files are hard-linked from the previous backup.

## Requirements

- Linux
- bash
- rsync
- a destination filesystem that supports hard links
  - for example: ext4, xfs, btrfs
- enough permissions to read all desired files
  - usually this means running as `root`

## Installation

Place the script somewhere suitable, for example:

```bash
sudo cp rsnap-backup /usr/local/sbin/rsnap-backup
sudo chmod 755 /usr/local/sbin/rsnap-backup
```

## Usage

```bash
sudo rsnap-backup /path/to/rsnap.ini
```

The script requires exactly one argument: the path to the INI configuration file.

## Configuration file format

Example:

```ini
[backup]
root=/home/username/backup

[sources]
path=/home/username/Music
path=/home/username/Documents

[exclude]
pattern=*.bak
pattern=*.tmp
pattern=*~
pattern=/home/*/.cache/*
pattern=/home/*/.local/share/Trash/*

[retention]
daily_days=30
keep_last_of_month=yes
```

## Configuration sections

### `[backup]`

Required.

- `root=...`
  - the backup destination root
  - this directory must already exist
  - if it does not exist, the script exits with an error

Example:

```ini
[backup]
root=/backup/myhost
```

### `[sources]`

Required.

Add one `path=...` line for each source file or directory to back up.

Examples:

```ini
[sources]
path=/etc
path=/home
path=/root/.bashrc
```

Notes:

- directories are backed up as directories
- regular files are backed up as regular files
- the source path `/` is intentionally not supported; list top-level paths explicitly instead

### `[exclude]`

Optional.

Add one `pattern=...` line for each rsync exclude pattern.

Examples:

```ini
[exclude]
pattern=*.bak
pattern=*.tmp
pattern=*~
pattern=/home/*/.cache/*
```

Notes:

- `*.bak` matches files ending in `.bak` anywhere in the tree
- exclude patterns are passed directly to `rsync`
- full-line comments beginning with `#` or `;` are allowed
- inline comments after values are not currently supported

### `[retention]`

Optional.

Supported keys:

- `daily_days=N`
- `keep_last_of_month=yes|no`

Example:

```ini
[retention]
daily_days=30
keep_last_of_month=yes
```

Behavior:

- keep all backups from the last 30 days
- for backups older than 30 days, keep only the most recent backup from each month
- delete all other old backups automatically after a successful backup run

## Backup layout

Example layout:

```text
/backup/myhost/
    2026-03-22_13-41-45/
    2026-03-23_13-41-45/
    logs/
```

Each run also writes a log file under `logs/`:

```text
/backup/myhost/logs/2026-03-23_13-41-45.log
```

## Restoring files

No special restore tool is required.

To restore a file, just copy it out of the desired snapshot.

Example:

```bash
cp /backup/myhost/2026-03-22_13-41-45/etc/hosts /etc/hosts
```

## Deleting old snapshots manually

Snapshots can be deleted with ordinary commands such as:

```bash
rm -rf /backup/myhost/2026-01-15_03-00-00
```

This is safe for the remaining snapshots because unchanged files are connected by hard links. Deleting one snapshot only removes that snapshot's directory entries. The file data remains as long as some other snapshot still links to it.

Snapshots should be treated as read-only.

## Hard link behavior

This script preserves two kinds of hard-link behavior:

1. Existing hard links inside a single backed-up source tree are preserved by `rsync -H`.
2. Unchanged files between snapshots are hard-linked by `--link-dest`.

One limitation is that hard links that cross between two separately listed source paths are not preserved as cross-source hard links, because each source is backed up in a separate `rsync` run.

## Notes and cautions

- Run as `root` if you need access to protected files.
- The destination root must already exist.
- The destination filesystem must support hard links.
- All snapshots for a given backup root must remain on the same filesystem.
- Do not edit files inside snapshot directories.
- Treat snapshots as read-only.
- Use test data first before trusting any backup script with important data.

## Example session

```bash
sudo rsnap-backup /etc/rsnap.ini
```

Possible output:

```text
[2026-03-22 13:41:45] starting backup: /home/username/backup/2026-03-22_13-41-45
[2026-03-22 13:41:45] latest prior backup: /home/username/backup/2026-03-21_13-30-00
[2026-03-22 13:41:45] backing up /home/username/Documents
[2026-03-22 13:41:45] backing up /home/username/Music
[2026-03-22 13:41:47] backup complete
[2026-03-22 13:41:47] rsync log: /home/username/backup/logs/2026-03-22_13-41-45.log
```

## Suggested future improvements

Possible enhancements include:

- dry-run mode
- per-config lock files
- verification mode
- email or summary reporting
- inline comment support in the INI parser
- snapshot metadata file
- automatic scheduling examples with `systemd`

## License

Add whatever license you prefer for this repository.
