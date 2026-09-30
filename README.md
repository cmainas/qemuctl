# qemuctl

`qemuctl` is a single bash script for running several qemu/KVM VMs from
cloud images on one host.

* **Images** are read-only base qcow2 files in `images/`.
* **VMs** are qcow2 overlays on an image in `vms/<name>/`, each with its own
  cloud-init seed (hostname = VM name), monitor/serial sockets, pidfile and
  **auto-allocated host ports**, so any number of VMs can run side by side.
* **Snapshots** are qcow2 checkpoints of a VM disk. A customised VM can also
  be flattened into a new base image with `image create-from`.

## Install

```sh
sudo apt install qemu-system-x86 qemu-utils cloud-image-utils socat
sudo usermod -aG kvm $USER      # optional: avoids sudo for qemu; log in again
./qemuctl doctor
```

Without the `kvm` group, `qemuctl` falls back to sudo: `sudo -g kvm` if your
sudoers rule allows a group (files stay owned by you), otherwise plain
`sudo` (qemu runs as root and `qemuctl` chowns the pidfile, sockets and log
back to you). `qemuctl doctor` tells you which mode applies.

## Quick start

```sh
./qemuctl image pull ubuntu-22.04                 # -> images/ubuntu-22.04.qcow2
./qemuctl vm create web -p 8080 --start           # ssh + guest 8080 forwarded
./qemuctl vm create db --mem 2G --cpus 1 --start  # second VM, different ports
./qemuctl vm list
./qemuctl vm ssh web                              # user ubuntu, password pass
./qemuctl vm console web                          # serial console, Ctrl-] detaches
./qemuctl vm scp web ./file :/tmp/                # ':' prefix = inside the VM
./qemuctl vm sync web push ~/work/urunc /home/ubuntu/work/urunc
./qemuctl vm sync web pull ~/work/urunc /home/ubuntu/work/urunc
./qemuctl vm stop web
```

Every `vm` command also works without the `vm` word: `./qemuctl ssh web`.

## Commands

| Command | What it does |
|---|---|
| `doctor` | Check dependencies and `/dev/kvm` access |
| `image list` | Show images, sizes and which VMs use them |
| `image pull <alias\|URL> [--name N] [--resize +10G]` | Download an image (aliases: `ubuntu-22.04`, `ubuntu-24.04`) |
| `image import <file> [--name N]` | Copy a local qcow2/raw file into the store |
| `image create-from <vm> <image> [--compress]` | Flatten a stopped VM's disk into a new base image (`--compress` is slower but ~2.5x smaller) |
| `image delete <image>` | Remove an image (refused while VMs back on it) |
| `vm create <name> [--image I] [--cpus N] [--mem 4G] [--disk +10G] [-p [host:]guest ...] [--ssh-port N] [--start]` | Create a VM. `-p 8080` picks a free host port; `-p 25778:8080` pins one |
| `vm start <name> [--dry-run]` | Boot in the background (`--dry-run` prints the qemu command) |
| `vm stop <name> [--force]` | ACPI shutdown, wait 60 s, then kill (`--force` kills now; avoid during a first boot, it can leave cloud-init half done) |
| `vm restart <name>` | Stop then start |
| `vm list` / `vm info <name>` | State, image, cpus, memory, host ports |
| `vm ssh <name> [-- cmd]` | SSH via the forwarded port |
| `vm scp <name> [scp opts] <src>... <dst>` | Copy files with scp. A path starting with `:` is inside the VM, e.g. `:/var/log/syslog` |
| `vm sync <name> push\|pull [local] [remote] [--delete] [--git] [-n] [--exclude P]` | rsync a directory between host and VM. `push` mirrors local to the VM including `.git`; `pull` is additive and skips `.git` unless `--git`. `--delete` removes stale files on the receiving side, `-n` is a dry run. One sync per VM at a time; log in `vms/<name>/sync.log` |
| `vm console <name>` | Attach to the serial console |
| `vm clone <src> <dst>` | Copy a stopped VM with a fresh seed and new ports |
| `vm delete <name> [--force]` | Remove a VM (`--force` stops it first) |
| `vm snapshot create <vm> [name]` | Checkpoint a stopped VM's disk |
| `vm snapshot list\|revert\|delete <vm> [name]` | Manage checkpoints |

## Configuration (environment)

| Variable | Default | Meaning |
|---|---|---|
| `QEMUCTL_HOME` | `~/vm_manager` | Where `images/` and `vms/` live |
| `QEMUCTL_PORT_BASE` / `QEMUCTL_PORT_RANGE` | `22000` / `1000` | Host port pool for auto-allocation |
| `QEMUCTL_DEFAULT_IMAGE` | only image present | Image used when `--image` is omitted |
| `QEMUCTL_DEFAULT_CPUS` / `QEMUCTL_DEFAULT_MEM` | `2` / `4G` | VM sizing defaults |
| `QEMUCTL_SSH_USER` / `QEMUCTL_PASSWORD` | `ubuntu` / `pass` | Cloud-init login |
| `QEMUCTL_SYNC_LOCAL` / `QEMUCTL_SYNC_REMOTE` | none | Default directories for `vm sync` when not given on the command line |

## Layout

```
images/<image>.qcow2         base images
vms/<name>/disk.qcow2        overlay (backing file = the image)
vms/<name>/seed.img          cloud-init NoCloud seed (user-data, meta-data next to it)
vms/<name>/vm.conf           IMAGE CPUS MEM SSH_PORT PORTS CREATED
vms/<name>/qemu.pid, monitor.sock, serial.sock, serial.log, sync.log
```
