+++
title = 'Easy Proxmox upgrade and VM migration without PBS'
date = 2024-07-06T00:00:00+01:00
draft = false
disableComments = true
+++
## Why now and why so much effort

Proxmox 7 is becoming EOL in July 2024 so you need to upgrade to a new OS. You can use this mandatory upgrade as an excuse to switch out your current hardware to something new, like me.

If you don't care about starting fresh and you have the option to do an in-place upgrade with Proxmox Backup Server, then certainly do it that way.

## Prerequisites

- USB flashdrive for a new Proxmox ISO (it hurts me to say this but avoid using Ventoy)
- machine running outdated Proxmox eg. Proxmox 7.x
- SSH access to PVE
- sudo installed and a sudo user (Proxmox doesn't have sudo installed by default)

## VM backup

To back up your VMs, SSH with a sudo user into PVE and run `sudo vzdump <VM ID> --compress <compression format> --storage <storage ID>` for every VM that you want to migrate. For example: `sudo vzdump 100 --compress zstd --storage local`.

If you have used Proxmox installer to partition the disk during installation and have not changed default backup dump directory, all your VM backups should be in `/var/lib/vz/dump`.

Now we need to get these files onto our machine. We can use 'scp' for this. We can do this using a single command (provided you have used the same compression format when backing up your VMs). Run `scp <user>@<host>:/path/to/backups/*.zst /path/to/backups/local`. For example: `scp mark@192.168.1.2:/var/lib/vz/dump/*.zst /home/mark/Desktop/proxmox-backups`. This will copy all files ending in '.zst' to a provided local directory.

## Install and configure Proxmox

Flash the USB with Proxmox ISO. Avoid using Ventoy as it has some problems with Proxmox at the time of writing this. There is not much to say here. Just follow the GUI installer. You can find the link [here](https://www.proxmox.com/en/proxmox-virtual-environment/get-started). Don't forget to setup SSH access to the server and install sudo.

## Copy the backups to the new OS

Here we can once again use `scp` - `scp /path/to/backups/local/*.zst <user>@<host>:/path/to/backups`. For example: `scp /home/mark/Desktop/proxmox-backups/*.zst mark@192.168.1.2:/var/lib/vz/dump/`. This will once again copy all files ending in '.zst' to a specified remote location.

## Run the VMs

You can do this with `sudo qmrestore` - `qmrestore <archive> <vmid>`. Run this command for each VM you want to run. For example: `sudo qmrestore vzdump-qemu-100-2024_07_05-23_47_10.vma.zst 100`.

I hope this helped you out:]