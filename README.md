# disk-tool

## WARNING

`disk-tool` can permanently destroy data, erase entire drives, overwrite the
wrong disk in seconds, and trigger destructive tests that cannot be undone.

Do not run this tool unless you fully understand:

- which device you are targeting
- which command you are running
- whether that command is destructive
- whether the target disk contains anything you may need later

If you select the wrong drive, the wrong partition, or the wrong erase mode,
data loss may be immediate and irreversible.

Use this tool carefully, verify device names before every run, and never test
dangerous commands on a machine or disk you are not prepared to wipe.

## Overview

`disk-tool` is a Debian/Ubuntu-focused shell utility for disk imaging,
inspection, benchmarking, testing, and erase workflows.

It is designed as a single portable command with:

- apt-based setup
- a simple public CLI
- strong destructive-operation warnings
- timestamped per-device logs
- install and update helpers
- bash completion support

## Target Platform

This project currently targets:

- Debian
- Ubuntu

The setup flow assumes `apt-get` and Debian/Ubuntu package names.

## Command Summary

```text
disk-tool --setup
disk-tool --install [--force] [--no-completion]
disk-tool --update [--force]
disk-tool --help
disk-tool --version

disk-tool doctor
disk-tool list
disk-tool removable
disk-tool info DEVICE
disk-tool benchmark DEVICE
disk-tool dd SOURCE TARGET
disk-tool drive
disk-tool drive DEVICE MODE
disk-tool completion bash
disk-tool completion install
```

## Safety Model

Many `disk-tool` operations are intentionally destructive.

Destructive actions include:

- `disk-tool dd`
- `disk-tool drive test`
- `disk-tool drive wipe`
- `disk-tool drive quick-wipe`
- `disk-tool drive full-wipe`
- `disk-tool drive burnin`
- `disk-tool drive discard`
- `disk-tool drive ata-erase`
- `disk-tool drive nvme-erase`
- `disk-tool drive shred-erase`

Safety behaviors:

- destructive commands show a large warning banner
- destructive commands print target device details before continuing
- destructive commands require typing the exact target path unless `--yes` is used
- destructive commands refuse mounted target devices by default
- `--force` can override the mounted-device refusal

`--yes` is intended for trusted automation or experienced use. It should be used
carefully.

## Installation And Setup

### One-shot setup

The main supported setup path is:

```bash
disk-tool --setup
```

This installs:

- the `disk-tool` launcher
- the full dependency set for all current commands
- bash completion, unless `--no-completion` is used

Packages installed by `--setup`:

- `curl`
- `util-linux`
- `coreutils`
- `pv`
- `hdparm`
- `smartmontools`
- `e2fsprogs`
- `fio`
- `nvme-cli`
- `time`

### Install from the script

If you already have the script downloaded:

```bash
./disk-tool --install
```

Use `--force` to replace an existing launcher:

```bash
./disk-tool --install --force
```

Skip completion install if needed:

```bash
./disk-tool --install --no-completion
```

### Updating

Update from the script you already have:

```bash
./disk-tool --update
```

Update from a remote raw URL:

```bash
export DISK_TOOL_RAW_URL="https://raw.githubusercontent.com/OWNER/REPO/main/disk-tool"
disk-tool --update
```

### Bash completion

Print the bash completion script:

```bash
disk-tool completion bash
```

Install bash completion:

```bash
disk-tool completion install
```

## GitHub-Friendly Install Examples

Example raw install pattern:

```bash
curl -fsSL https://raw.githubusercontent.com/OWNER/REPO/main/disk-tool -o disk-tool
chmod +x disk-tool
sudo ./disk-tool --install
```

Example raw update pattern:

```bash
export DISK_TOOL_RAW_URL="https://raw.githubusercontent.com/OWNER/REPO/main/disk-tool"
disk-tool --update
```

Replace `OWNER/REPO` with the actual GitHub repository path.

## Command Reference

### `disk-tool doctor`

Shows environment and dependency status.

It reports:

- launcher status
- configured remote raw URL
- key binary locations
- missing dependency binaries

Example:

```bash
disk-tool doctor
```

### `disk-tool list`

Lists available disks using `lsblk`.

The output includes:

- path
- size
- device type
- transport
- removable flag
- rotational flag
- model
- serial

Example:

```bash
disk-tool list
```

### `disk-tool removable`

Shows likely removable disks such as USB storage or MMC devices.

Example:

```bash
disk-tool removable
```

### `disk-tool info DEVICE`

Shows a non-destructive device summary.

It combines:

- `lsblk` device details
- partition table overview
- filesystem signatures
- model/serial identity
- SMART info and health when available

Example:

```bash
disk-tool info /dev/nvme0n1
```

### `disk-tool benchmark DEVICE`

Runs a read-only benchmark against a block device using `fio`.

Notes:

- this is not destructive
- it can heavily load the selected disk
- it writes logs to the per-device log directory
- it currently runs a timed read test for 30 seconds

Example:

```bash
sudo disk-tool benchmark /dev/nvme0n1
```

### `disk-tool dd SOURCE TARGET`

Writes `SOURCE` to `TARGET` using:

- `bs=4M`
- `iflag=fullblock`
- `status=progress`
- `conv=fsync`

Notes:

- this command requires root
- `SOURCE` must exist
- `TARGET` should be a full device path such as `/dev/sda` or `/dev/nvme0n1`
- mounted targets are refused unless `--force` is used
- destructive confirmation requires the exact target path unless `--yes` is used

Example:

```bash
sudo disk-tool dd ~/Downloads/debian.iso /dev/sda
```

### `disk-tool drive`

Shows drive help and the list of available drive modes.

Example:

```bash
disk-tool drive
```

### `disk-tool drive DEVICE MODE`

Runs drive-management actions against a block device.

Important input detail:

- `DEVICE` should always be a full device path such as `/dev/sda` or `/dev/nvme0n1`

To quickly inspect available targets before running a drive command:

```bash
disk-tool list
disk-tool removable
```

Examples:

```bash
sudo disk-tool drive /dev/sda serial
sudo disk-tool drive /dev/sdb smart
sudo disk-tool drive /dev/sdc test
sudo disk-tool drive /dev/sdc burnin
sudo disk-tool drive /dev/sdd quick-wipe
sudo disk-tool drive /dev/sdd ata-erase
sudo disk-tool drive /dev/nvme0n1 nvme-erase
```

#### Drive modes

##### `serial`

Shows identifying information for the selected device.

Behavior:

- SATA/SAS-style devices use `hdparm -I`
- NVMe devices use `nvme id-ctrl`

Examples:

```bash
sudo disk-tool drive /dev/sda serial
sudo disk-tool drive /dev/nvme0n1 serial
```

##### `smart`

Shows SMART info, attributes, and health status using `smartctl`.

Example:

```bash
sudo disk-tool drive /dev/sdb smart
```

##### `test`

Runs:

```bash
/usr/bin/time --verbose badblocks -wvs /dev/DEVICE
```

Notes:

- destructive
- mounted targets are refused unless `--force` is used

Example:

```bash
sudo disk-tool drive /dev/sdc test
```

##### `wipe`

Runs a single-pass shred wipe:

```bash
/usr/bin/time --verbose shred --verbose --iterations=1 /dev/DEVICE
```

Notes:

- destructive
- mounted targets are refused unless `--force` is used

Example:

```bash
sudo disk-tool drive /dev/sdd wipe
```

##### `quick-wipe`

Runs a faster signature-oriented wipe flow:

- `wipefs -a`
- zero the first 16 MiB
- zero the last 16 MiB

This is useful when you want to quickly remove partitioning and common
filesystem signatures without doing a full multi-pass wipe.

Example:

```bash
sudo disk-tool drive /dev/sdd quick-wipe
```

##### `full-wipe`

Runs:

```bash
/usr/bin/time --verbose shred --verbose --iterations=7 --zero /dev/DEVICE
```

Example:

```bash
sudo disk-tool drive /dev/sdd full-wipe
```

##### `burnin`

Runs a destructive validation sequence with logs:

- `badblocks`
- `fio`
- `smartctl --health`

Notes:

- destructive
- mounted targets are refused unless `--force` is used
- logs are written for each run
- `fio` is configured for a one-hour timed run

Example:

```bash
sudo disk-tool drive /dev/sde burnin
```

##### `discard`

Runs:

```bash
blkdiscard -f /dev/DEVICE
```

This is intended for supported SSD and NVMe devices.

Example:

```bash
sudo disk-tool drive /dev/nvme0n1 discard
```

##### `ata-erase`

Runs ATA secure erase using `hdparm`.

Command flow:

```bash
hdparm --security-set-pass NULL /dev/DEVICE
hdparm --security-erase NULL /dev/DEVICE
```

Notes:

- destructive
- mounted targets are refused unless `--force` is used
- valid for non-NVMe devices

Example:

```bash
sudo disk-tool drive /dev/sdd ata-erase
```

##### `nvme-erase`

Runs:

```bash
nvme format /dev/DEVICE --ses=1
```

Notes:

- destructive
- mounted targets are refused unless `--force` is used
- valid only for NVMe devices

Example:

```bash
sudo disk-tool drive /dev/nvme0n1 nvme-erase
```

##### `shred-erase`

Runs:

```bash
/usr/bin/time --verbose shred --verbose -z -n 3 /dev/DEVICE
```

Notes:

- destructive
- mounted targets are refused unless `--force` is used

Example:

```bash
sudo disk-tool drive /dev/sdf shred-erase
```

## Logging

Drive and benchmark operations create timestamped log directories automatically.

Default log location:

```text
./disk-tool-logs
```

Example structure:

```text
disk-tool-logs/
  sda/
    20260331-184500/
      smart.log
  nvme0n1/
    20260331-185000/
      benchmark.log
      burnin.log
      fio.log
      badblocks.txt
```

You can override the log location with:

```bash
export DISK_TOOL_LOG_DIR="$HOME/disk-logs"
```

## Typical Workflows

### Inspect a system before making changes

```bash
disk-tool list
disk-tool removable
disk-tool info /dev/nvme0n1
```

### Flash a bootable image

```bash
disk-tool --setup
sudo disk-tool dd ~/Downloads/ubuntu.iso /dev/sda
```

### Benchmark a drive without destroying data

```bash
sudo disk-tool benchmark /dev/nvme0n1
```

### Burn in a drive and keep logs

```bash
sudo disk-tool drive /dev/sdc burnin
```

### Quickly clear signatures on a device

```bash
sudo disk-tool drive /dev/sdd quick-wipe
```

### Secure erase an NVMe SSD

```bash
sudo disk-tool drive /dev/nvme0n1 discard
sudo disk-tool drive /dev/nvme0n1 nvme-erase
```

## Exit Codes

`disk-tool` currently uses these main exit codes:

- `0` success
- `1` general error, invalid arguments, missing device, unsupported mode, or canceled confirmation

More specialized non-zero exit codes can be added later, but today the tool is
primarily oriented around readable terminal messages rather than a large exit
code matrix.

## Troubleshooting

### `disk-tool: command not found`

Run:

```bash
./disk-tool --install
```

### Missing dependency errors

Run:

```bash
disk-tool --setup
```

### Permission denied or root required

Most hardware-changing operations require elevated privileges.

### Wrong device format passed to `drive`

Use:

```bash
sudo disk-tool drive /dev/sda smart
```

Not a bare short name.
