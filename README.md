
# VirtualBox Clone Backup
VirtualBox Clone Backup (VBCB) is a script for creating Oracle VirtualBox VM backups in Windows host environments with as little downtime as possible.

VBCB is a fork of the excellent https://github.com/niro1987/VirtualBox-Backup (VBB) project, the key being the use of `clonevm` command instead of `robocopy` to create backups.

The main advantage of using `clonevm` is that fewer files and no live data are copied.  `clonevm` is also configured to create a new UUID for the backup, which means the backup is immediately bootable without having to unregister the original VM.

Additional changes from VBB include:
* Re-organization of backup directory by sub-directory based on VM names and timestamps.
* Added `--backupmode start` options which starts the VM immediately after snapshotting.

## How I Like to Use VBCB
VBB is great, but one of the issues that I have with it is that in order to ensure a stable backup of a live VM, the VM had to be taken offline for the duration of the backup.

When I make a backup of a live VM, I first shutdown it down then execute a script that launches VBCB with the `--backupmode start` option.  With this option enabled, VBCB does the following:
1. Takes a snapshot of the VM.
2. Starts a clone of the VM from the snapshot.
3. Restarts the VM while the clone is executing.

Because the snapshot is taken while the VM is offline, it is both fast and stable.  And, because the clone is made from a snapshot, there is nothing preventing the VM from restarting immediately.  Backups of a live VM take only as long to shutdown and restart.  

## Table of Contents
- [VirtualBox Clone Backup](#virtualbox-clone-backup)
  - [Installation](#installation)
  - [Usage](#usage)
    - [Backup Dir](#backup-dir)
      - [Snapshot Only](#snapshot-only)
    - [Backup Mode](#backup-mode)
    - [Prefix/Suffix](#prefixsuffix)
    - [Include/Exclude](#includeexclude)
    - [Compress](#compress)
    - [Keep](#keep)
    - [Stack](#stack)
  - [Restoring Backups](#restoring-backups)
  - [Advanced Usage](#advanced-usage)
    - [Grandfather-Father-Son Rotation](#grandfather-father-son-rotation)
  - [Changes](#changes)
  - [Credits](#credits)

## Installation

1. Clone or copy this repository to the desired location.
2. Edit and rename *(optional)* **Example Start.bat** according to your needs. See below [Usage](#Usage)
3. Create a basic task to periodically start **Example Start.bat** *(or whatever you named it)* with [Task Scheduler](https://www.google.com/search?q=Windows+Task+Scheduler&oq=Windows+Task+Scheduler).

Using *Example Start.bat* from Task Scheduler makes editing the parameters a bit more user friendly and easier to duplicate.

## How It Works
VBCP starts by taking a snapshot of a target vm.  The `backupmode` param lets you choose between taking live snapshots or snapshots from stopped or saved states.

If `backupdir` is specified, the snapshot is also cloned to the specified directory as follows:

	<backupdir>/
	├─ <vm>/
	│  ├─ <prefix><vm_name>-YYYY.MM.DD-HH.MM<suffix>/
	│  │  ├─ <prefix>YYYY.MM.DD-HH.MM<suffix>.vbox
	│  │  ├─ <prefix>YYYY.MM.DD-HH.MM<suffix>.vdi
	│  │  ├─ Snapshots/

Where `<backupdir>`, `<vm>`, `<prefix>` and `<suffix>` are specified by options.

Additional features:
1. Save or remove the backup snapshot in the target VM.
2. Set the number of backups to keep.
3. Compress backups.
4. Include or exclude source VMs and backup more than one VM at a time.
5. Stop or save VM state before taking snapshots and resume them afterward.
6. Take live snapshots.
7. Clone snapshots while the VM is running.

## Restoring a Backup
Since each VBCP backup is  created using `vboxmanage clonevm` with a new UUID, each backup is immediately available to be added in VirtualBox Manager by tapping `Machine > Add...`.  

## Usage

If you do not pass a parameter it will revert to it's default behavior as documented below.

### Backup Dir

```text
[ --backupdir PATH ]
```

Pass this parameter along with a valid path to set the target backup directory. A sub-directory is automatically created for each VM. Your can use Windows default variables like `%USERPROFILE%` and `%ONEDRIVE%`. The custom variable `%_CURRENTDIR%` will set the target folder to where you save the `.bat` files.

#### Snapshot Only

Leaving the `backupdir` parameter out will create a snapshot of the VM without copying any files. This setting automatically sets [Backup Mode](#backup-mode) to `snapshot` and enables [Stack](#stack). [Keep](#keep) is currently not supported in combination with this setting. [Compress](#compress) does not apply and is ignored.

| Parameter                     | Description                           |
| ----------------------------- | ------------------------------------- |
| `--backupdir "%_CURRENTDIR%"` | Parent folder, see above description. |
| `--backupdir "%USERPROFILE%"` | C:\Users\YourUsername                 |
| `--backupdir "C:\Backup"`     | C:\Backup                             |

### Backup Mode

```text
[ --backupmode { acpipowerbutton | savestate | snapshot | start } ]
```

In order to successfully create a backup, the VM needs to be in a stable (not changing) state. To reduce downtime, a snapshot is created and the VM is restarted (if it was running in the first place).

To restore a backup you simply copy/extract the files to your desired location, add (add, not new) VM to OracleBox and restore the latest snapshot. Be aware that you will not be able to restore a backup while the original VM still exists in the same instance of VirtualBox because the drives will have identical UUID's.

| Parameter                      | Description                                                                                                                                                                                                                                                                              |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--backupmode acpipowerbutton` | The VM is completely shut down and boots normally after the snapshot is created. Not ideal if login is required after boot. Booting a restored backup is like normal booting the VM. |
| `--backupmode savestate`       | The VM's state is frozen and saved, VM resumes normally after the snapshot is created. Not all operating systems can handle this *gap* in time. Booting a restored backup is like unfreezing time, the same *gap* applies. You might need to restart your VM to fix any time gap issues. |
| `--backupmode snapshot`        | The VM is saved in a live snapshot without any downtime. Booting a restored backup is as if the VM experienced a power failure. It the best suboptimal solution to prevent downtime.|
| `--backupmode start`           | The VM is started immediately after snapshotting.|

### Prefix/Suffix

```text
[ --prefix PREFIX ]
[ --suffix SUFFIX ]
```

Each backup is saved to a subfolder inside the [target folder](#backup-dir) named after the VM. The backup is named `[prefix ]YYYY.MM.DD-HH24.MM[ suffix]`. Pass one or both parameters to append an additional string to the backup name.

| Parameter                     | Description                                      |
| ----------------------------- | ------------------------------------------------ |
| `--prefix "Automated Backup"` | Prefix the backup name with `"Automated Backup"` |
| `--suffix="Daily"`            | Suffix (append) the backup name with `"Daily"`   |

### Include/Exclude

```text
[ --include VM-Name ]
[ --exclude VM-Name ]
```

Set one of the above parameters to exclude or explicitly include a single VM from backup. Does not accept wildcards and is case sensitive.

Explicitly *excluding* a single VM will still run backups for all other VMs. Explicitly *including* a single VM will exclude all other VMs.

| Parameter            | Description                                                                                |
| -------------------- | ------------------------------------------------------------------------------------------ |
| `--include "Remi"`   | Will include only the VM named `Remi` from the backup rotation and ignores the rest.       |
| `--exclude "Not Me"` | Will exclude only the VM named `Not Me` from the backup rotation but does backup the rest. |

### Compress

```text
[ --compress ]  [ -1 - 9 ]
```

***!!!*** In order to enable data compression you need to install [7-Zip](https://www.7-zip.org/) to the default path `C:\Program Files\7-Zip\7z.exe`. To disable compression, leave this parameter out or explicitly set it to `-1`.

The VM Backup Files can be compressed to a single 7-Zip file to save some diskspace and to make them easier to move around.

| Parameter            | Description                                                                                                    |
| -------------------- | -------------------------------------------------------------------------------------------------------------- |
| `--compress -1`      | *(default)* Disable compression.                                                                               |
| `--compress 0`       | No compression rate. While this does not actually reduce filesize, it does reduce the backup to a single file. |
| `--compress [1 - 9]` | Set compression level: 1 (fastest) ... 9 (ultra).                                                              |

### Keep

```text
[ --keep  { 0 | 1 | 2 | ... | N } ]
```

Delete old backups with the same prefix and/or suffix and retaines the last `[x]`. If no [Prefix and/or Suffix](#prefixsuffix) is set, all files and folders in the VM's backup subfolder are validated and possibly removed.

| Parameter    | Description                               |
| ------------ | ----------------------------------------- |
| `--keep 0`   | *(default)* No cleanup. Keep all backups. |
| `--keep N`   | Retain the `N` latest created backups.  |

### Stack

```text
[ --stack ]
```

A snapshot is always created before the files are copied. This uses some disk space because VirtualBox saves the new state on top of the latest snapshot. To save disk space, the snapshot is deleted after the backup is created (unless you're using [Snapshot Only](#snapshot-only)). Add this flag to retaun the snapshots, stacking each snapshot on top of the previous one (uses a lot of disk space).

| Parameter | Description           |
| --------- | --------------------- |
| `--stack` | Retain all snapshots. |

## Advanced Usage

### Grandfather-Father-Son Rotation

[Wikipedia](https://en.wikipedia.org/wiki/Backup_rotation_scheme)

You will need to create and schedule multiple **Example Start.bat** files, one for each generation. Make sure you use a different prefix, suffix or even target folder for each generation to prevent unintended deletion of backups.

| Rotation                  | Parameters | Description                                                     |
| ------------------------- | ---------- | --------------------------------------------------------------- |
| `Daily Son.bat`           | `--keep=2` | Scheduled to run at `02:00` daily.                              |
| `Weekly Father.bat`       | `--keep=4` | Scheduled to run at `02:15` on every Monday of every week.      |
| `Monthly Grandfather.bat` | `--keep=3` | Scheduled to run at `02:30` on the first Monday of every month. |

By the end of June 2020 it would look like this.

| *Day*       | 1-4 | 1-5 | 1-6 | 8-6 | 15-6 | 22-6 | 28-6 | 29-6 | 30-6 |
| ----------- | --- | --- | --- | --- | ---- | ---- | ---- | ---- | ---- |
| Son         | -   | -   | -   | -   | -    | -    | X    | -    | X    |
| Father      | -   | -   | -   | X   | X    | X    | -    | X    | -    |
| Grandfather | X   | X   | X   | -   | -    | -    | -    | -    | -    |

## Compatibility
Should work with VB 6.1.x and VB 7.0.x.  I'm personally using it with 6.1.40+ and 7.0.14+.
 
## Change Log
| Date | Changes
|------------|----------------------------------------------------------------
| 2023-03-05 | Initial release with support for VB 6.1.x.
| 2024-02-04 | Update and clean up with support for VB 7.0.14.
| 2025-06-08 | Update README.md.

## Credits
[VirtualBox.org](https://virtualbox.org) team for obvious reasons.
[niro1987](https://github.com/niro1987) for kindly developing and making [VirtualBox-Backup](https://github.com/niro1987/VirtualBox-Backup) available.

<!--stackedit_data:
eyJoaXN0b3J5IjpbMjExMDg4MjMyMSwtNzc3OTg2NzQ5XX0=
-->
