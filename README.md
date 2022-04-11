This is a simple shell script which wraps calls to `rdiff-backup` with the starting and stopping of `systemd` units.

Use case
========
I find it useful to keep the system's backup destination offline (at least from the point of view of the backup source, i.e. unmounted) most of the time, significantly reducing the risk of unauthorized access to or accidental corruption of the backups. This makes particular sense in the case the backup destination is local to the backup source.

The script will first connect the system to the backup destination through `systemd` (e.g. mount the backup destination disk), then, if successful, perform the backup and optionally clean up old backup increments, and finally disconnect (e.g. unmount) the backup destination.

Development status / quality
============================
This is a simple script intended only for a personal, rather constrained use case.
As such it comes on an "as-is" basis and without any guarantee whatsoever (see `LICENSE.txt` for details).

It nevertheless features minimal logging and error-handling. It is, except for the use of function-`local` variables (which can easily be made global if needed), POSIX-compliant.

Usage
=====
`rdiff-backup-wrapper [options] --src backup_source --dst backup_destination -- [rdiff_backup_options]`

`rdiff-backup-options` are transferred as-is to `rdiff-backup` and as such any option accepted by `rdiff-backup` is acceptable here.

Options
-------
- `--prerun systemd-unit1[,systemd-unit2,...]`: Comma-separated list of systemd units to be started before running `rdiff-backup`.
Each unit will be started in the order it is listed. Any failure to start a unit will abort the backup.
- `--postrun systemd-unit3[,systemd-unit1,...]`: Comma-separated list of systemd units to be stopped before exiting.
Each unit will be stopped in the order it is listed. Units are always stopped, whether the backup ran successfully or not.
- `-l|--label backup_label`: An arbitrary label to help identify this instance of the program.
The label, if specified, will be mentionned in logs and user notifications.
- `-r|--remove-backup-older-than rdiff-backup_time_interval`: if specified, backup increments in the backup destination older than `rdiff-backup_time_interval` will be deleted.
Older backups are deleted after running `rdiff-backup` and irrespective of the result of the backup (note, however, that `rdiff-backup` only deletes increments, and will always leave at least the latest full backup in place). However, they are not deleted if the startup of any systemd unit failed.
The format is the same as the `time_interval` argument to `rdiff-backup`'s `--remove-older-than` option (e.g. `1W` for 1 week, `3M` for 3 months, `1B` for 1 backup increment, etc).

Contents
========
`rdiff-backup-wrapper`
----
Main script which starts systemd units, runs `rdiff-backup`, cleans up old backup increments and finally stops systemd units.
It logs its output, including errors, to `logger`'s `local7` facility with the `rdiff-backup-wrapper` tag. Under most systemd default configurations, latest logs can be consulted by running `journalctl -e -t rdiff-backup-wrapper`.

`cron/cron-backup`
----
Small helper script to call `rdiff-backup-wrapper` from a cron task.
It will call `rdiff-backup-wrapper` with appropriate `ionice`, `nice` and `nocache` settings to reduce the load on the system.

It requires a `-i|--identifier` parameter which will ensure it runs all calls with the same identifier sequentially (it will wait for other instances with the same identifier to exit before proceeding to call `rdiff-backup-wrapper` ; if this doesn't happen before a hard-coded timeout -- currently 10 minutes, the script will simply exit without proceeding further). It is recommended to specify the same identifier for at least backups to a same destination, or backups starting / stopping some common systemd units, to prevent them from running concurrently.

```
cron-backup --identifier jobid -- rdiff-backup-wrapper_parameters
```

`notify-send-from-root`
----
Helper script which allows to `notify-send` desktop notifications to users when running as `root`.
It will notify all open sessions.
Despite `rdiff-backup-wrapper` being often run by `root` (notably as a cron job), it can be valuable in certain environments to have current desktop users notified of any backup error.

Installation
============
It is suggested to simply copy each script to `/usr/local/bin` and `/usr/local/sbin`.
One can then call `rdiff-backup-wrapper` like so (case of backing up to an encrypted local disk):
```
/usr/local/bin/rdiff-backup-wrapper --prerun systemd-cryptsetup@backupsystem,mnt-backup-system.mount --postrun mnt-backup-system.mount,systemd-cryptsetup@backupsystem --src / --dst /mnt/backup/system/ --label "System backup" -r 1M -- --include-globbing-filelist /usr/local/etc/rdiff-backup/system_backup_filelist
```
Or as a cron job, for example with the following command:
```
/usr/local/bin/cron/cron-backup --identifier system -- --prerun systemd-cryptsetup@backupsystem,mnt-backup-system.mount --postrun mnt-backup-system.mount,systemd-cryptsetup@backupsystem --src / --dst /mnt/backup/system/ --label "System backup" -r 1M -- --include-globbing-filelist /usr/local/etc/rdiff-backup/system_backup_filelist
```

Other environments
==================
[`rdiff-backup-wrapper-win`](https://github.com/desseim/rdiff-backup-wrapper-win) is the equivalent project for Windows environments, written in PowerShell.
