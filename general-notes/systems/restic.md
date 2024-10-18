# Setting up Restic backup

After setting up an external server (such as for a [Hetzner storage box](/general-notes/systems/hetzner.md)), edit your `~/.ssh/config` to make these options simpler:

```
Host backup
  HostName u1234.your-storagebox.de
  User u1234
  Port 23
  ServerAliveInterval 60
  ServerAliveCountMax 240
```

where the last two are options recommended on [this page](https://restic.readthedocs.io/en/latest/030_preparing_a_new_repo.html#sftp).

Then create a new backup repo:

```bash
$ restic -r sftp:backup:/restic/aero init
enter password for new repository:
enter password again:
Fatal: create repository at sftp:backup:/restic/aero failed: sftp: "Failure" (SSH_FX_FAILURE)
```

This is because we don't have write access to `/restic` on the server. Instead, we specify:

```bash
$ restic -r sftp:backup:/home/restic/aero init
enter password for new repository:
enter password again:
created restic repository 26719cd1de at sftp:backup:/home/restic/aero

Please note that knowledge of your password is required to access
the repository. Losing your password means that your data is
irrecoverably lost.
```

We try setting

```bash
$ export RESTIC_REPOSITORY=sftp:backup:/home/restic/aero
$ export RESTIC_PASSWORD=...
```

Now we can check for snapshots, which should be empty:

```bash
$ restic snapshots
repository 26719cd1 opened (version 2, compression level auto)
created new cache in C:\Users\Amos Ng\AppData\Local\restic
```

We create `Documents\restic\includes.txt` with the list of directories we want to back up:

```
Documents
Videos
Music
Pictures
```

and we create `Documents\restic\excludes.txt` with the list of directories we want to exclude:

```
Documents/projects/**/target
Documents/**/node_modules
```

Next, we run the restic backup command:

```powershell
$ restic backup --files-from .\Documents\restic\includes.txt --exclude-file .\Documents\restic\excludes.txt --verbose
```

It's okay if you kill the process, or if the network times out. You can run the same command again, and Restic will pick up where it left off.

## External hard drive

To back up to an external hard drive, do

```
$ restic -r F:\restic\Aero init
created restic repository 956886e321 at F:\restic\Aero

Please note that knowledge of your password is required to access
the repository. Losing your password means that your data is
irrecoverably lost.
$ restic -r F:\restic\Aero backup --files-from .\Documents\restic\includes.txt --exclude-file .\Documents\restic\excludes.txt --verbose
open repository
repository 956886e3 opened (version 2, compression level auto)
created new cache in C:\Users\Amos Ng\AppData\Local\restic
lock repository
no parent snapshot found, will read all files
load index files

start scan on [Documents ...]
start backup on [Documents ...]
scan finished in 14.672s: 134559 files, 80.842 GiB
[0:21] 0.34%  107 files 282.363 MiB, total 134559 files 80.842 GiB, 0 errors ETA 1:43:38
...
```

## Windows sleep

In Settings, under "System > Power & Battery > Screen and Sleep", even when you set "When plugged in, put my device to sleep after" to "Never", the computer still goes to sleep after a while, causing the backup process to stop. We see from David Ford's eventual response to [this post](https://answers.microsoft.com/en-us/windows/forum/all/windows-10-and-11-power-settings-sleep-never-yet/830af0e5-0291-4cfd-8268-a2ac9e9411e1) that we are to follow [these instructions](https://www.tenforums.com/tutorials/72133-add-system-unattended-sleep-timeout-power-options-windows.html) and run in an elevated command prompt:

```
$ REG ADD HKLM\SYSTEM\CurrentControlSet\Control\Power\PowerSettings\238C9FA8-0AAD-41ED-83F4-97BE242C8F20\7bc4a2f9-d8fc-4469-b07b-33eb785aaca0 /v Attributes /t REG_DWORD /d 2 /f
```

Which enables us to then go to the "Control Panel > Hardware and Sound > Power Options > Edit Plan Settings" and click "Change advanced power settings" to see "Sleep > System unattended sleep timeout" and set the plugged in value to "0", because according to [this documentation](https://admx.help/?Category=Windows_10_2016&Policy=Microsoft.Policies.PowerManagement::UnattendedSleepTimeOutAC):

> If you specify 0 seconds, Windows does not automatically transition to sleep.

## Restoring from backup

Set up the repository and password as before, and then run:

```bash
$ restic snapshots  
repository 26719cd1 opened (version 2, compression level auto)
created new cache in /home/amos/.cache/restic
ID        Time                 Host             Tags        Paths
--------------------------------------------------------------------------------------
dc4b9667  2024-02-24 10:43:20  DESKTOP-N5K32L1              C:\Users\Amos Ng\Documents
                                                            C:\Users\Amos Ng\Music
                                                            ...

1e657c35  2024-02-29 13:45:12  DESKTOP-N5K32L1              C:\Users\Amos Ng\Documents
                                                            C:\Users\Amos Ng\Music
                                                            ...
--------------------------------------------------------------------------------------
```

If you run into this error:

```bash
$ restic snapshots
Fatal: Fatal: config cannot be loaded: unsupported repository version
```

You can update restic with

```bash
$ sudo restic self-update
writing restic to /usr/bin/restic
find latest release of restic at GitHub
latest version is 0.16.4
download SHA256SUMS
download SHA256SUMS.asc
GPG signature verification succeeded
download restic_0.16.4_linux_amd64.bz2
downloaded restic_0.16.4_linux_amd64.bz2
saved 24539136 bytes in /usr/bin/restic
successfully updated restic to version 0.16.4
```

Then, you can check the files in a snapshot:

```bash
$ restic ls latest | less
```

Restore one of them with

```bash
$ restic restore latest --path "/Documents/Some Backed Up Folder" --target ~/Downloads/restoration
repository 26719cd1 opened (version 2, compression level auto)
...
```

If you encounter something such as

```bash
$ restic -r "$RESTIC_HD/Aero" ls latest --path "/Pictures"
repository 956886e3 opened (version 2, compression level auto)
[0:00] 100.00%  16 / 16 index files loaded
snapshot filter (Paths:[/Pictures] Tags:[] Hosts:[]): no snapshot found
```

it's because you have to do this instead:

```bash
$ restic -r "$RESTIC_HD/Aero" ls latest:/Pictures
```

External disks from Windows would be prefixed by their drive letter. For example, to restore `D:\Zotero`:

```bash
$ restic -r "$RESTIC_HD/Aero" restore latest:/D/Zotero --target ~/Zotero
```

If you have multiple hosts backing up to the same repo, you can filter by host:

```bash
$ restic -r "$RESTIC_HD/Aero" restore latest --host DESKTOP-N5K32L1 --target ~/Downloads/restoration
```

If you try to restore a single file, you get

```bash
$ restic restore latest:/Documents/restic/includes.txt --host Amoss-MacBook-Air.local --target ~/Documents/restic/includes.txt
repository 26719cd1 opened (version 2, compression level auto)
[0:01] 100.00%  11 / 11 index files loaded
path Documents/restic/includes.txt: not a directory
```

As such, you may want to restore the entire directory to a different location and pick out the bits you want:

```bash
$ restic restore latest:/Documents/restic/ --host Amoss-MacBook-Air.local --target ~/Downloads/restic/
$ cp ~/Downloads/restic/includes.txt ~/Documents/restic
```

## Pruning backups

To keep only some backups, run

```bash
$ restic forget --keep-hourly 24 --keep-daily 7 --keep-weekly 4 --keep-monthly unlimited --prune
```
