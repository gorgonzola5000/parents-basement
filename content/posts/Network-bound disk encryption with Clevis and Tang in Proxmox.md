+++ 
draft = false
date = 2025-11-20T23:17:58+01:00
title = "Network-bound disk encryption with Clevis and Tang in Proxmox"
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
disableComments = true
+++

## All of the code

While my infra is constantly changing and this guide might not be up to date once the infra evolves, you can check out the code by looking at the most recent commit I am on as of time of writing this - [link to the commit](https://gitlab.com/starfruit6000/homelab/-/commit/1d8de954d71c676075dd6d987a91c3a31b2ea66f). I used this version just today to set up the local environment so it should be ok

## Why

Having run Proxmox for quite some time now, I have become fed up with it.

After running way too much pointless stuff to make my other stuff more secure:

- like setting up the aforementioned FreeIPA to act as my private CA to have my own certificates for all of my services to not have to deal with Let's Encrypt's very low thresholds
- like running a 3-node FreeIPA cluster on AlmaLinux. Because Kerberos authentication decided to stop functioning on the single node one.
- like configuring my domain and adding all of my VMs and bare-metal hosts, to have in-transit encryption for my NFSv4 file shares
- and setting up LDAP to have all of my users (me and my service account because why not) and have the same password to all of my self-hosted services
only then to realize that Proxmox Backup Server, which I relied on for backups, USES A DIFFERENT IMPLEMENTATION OF LDAP THAN THE PROXMOX HYPERVISOR AND THE PBS REFUSING TO WORK WITH FREEIPA. I have decided to ditch this perl script glued piece of shit in favor of more sophisticated tools like Kubernetes.

After hours of research I found my Kubernetes distribution - Talos. One caveat though, I wanted to have full disk encryption. Well, at least for my root. If I am ever robbed, I will have enough of things on my mind. I don't want to bother myself with thinking about someone having access to my data, maybe even passwords.

Fortunately, Talos offers disk encryption. However, it is extremely limiting. It's either:

- static passphrase which gets saved in plaintext on one of the unencrypted system partitions... I am not even going to comment on that
- TPM2 which needs secureboot which I don't have and I don't want to lose access to all of my data because my motherboard dies
- KMS which uses a proprietary protocol. Proprietary as in custom made and FOSS for Talos. This would require me to run something like OpenBAO/Hashicorp Vault which has huge resource requirements and would make a great chicken-and-egg problem for me in setting up my infrastructure. I thought that Talos was all about being the EDGE solution. Clevis/Tang uses nearly 0 resources...
- random string based on node UUID which is not that much different from the statis passphrase one

All of the above are useless for me. They all want the Talos cluster to decrypt itself automagically without any human interaction. I get that. But not with these kinds of trade-offs. I want to input the passphrase myself. Well, I don't, because my servers are headless. I want to use NBDE with Clevis and Tang. I don't want to reinvent the wheel. Why does this need to be so complicated?

So it was either:

- develop a custom solution and maybe try to get it upstream into Talos, adding Clevis as a new way to decrypt system disks or
- virtualizing the node in Proxmox which I hate because it's going to cause problems with using the GPU in the VM. I also come to hate Proxmox

I am gonna work on the first option once I don't have anything else to do in my free time but for the time being I am gonna set up Devian Trixie, install Proxmox on top of it and then set up NBDE.

## Standard operation - overview

The whole workflow of me decrypting my stuff should be like this:

- my always-on server decides to reboot or something - it needs some kind of human intervention
- I open up my laptop (disk encrypted duh) on the local network and bring up Tang
- Clevis running in initramfs sends requests to the Tang server and hopefully decrypts the system disk
- profit
- well, if I lose access to Tang keys or anything, I can connect the display and input an emergency passphrase. LUKS2 offers 32 keyslots so that's more than enough

## How to set this thing up - overview

- create a Tang server on my laptop
- create a preseeded Debian .iso so that I only need to click enter during installation - I hate installing stuff and I would rather automate it hence the preseed
- install Debian - one click install. Maybe a few more if there are more disks available but more on that later
- install Proxmox on top of it
- configure the host with Ansible
- bind Clevis to the Tang server
- add emergency passphrase
- delete temporary bootstrap passphrase
- profit

## How to set this thing up - the long version

### Debian preseed

I found some shell script online but I lost the link. Anyway, it grabs a Debian .iso, preseeds it with the preseed.cfg file and then creates the custom .iso. Neat.

```shell

#!/bin/sh

set -e

ISO="$1"
NEW_ISO="$2"
WD="$(mktemp -d)"
OLD_PWD="$(pwd)"
echo $ISO
echo $WD
echo $OLD_PWD
7z x -o"$WD" "$ISO"

cd "$WD"

gunzip install.amd/initrd.gz
cp "$OLD_PWD/preseed.cfg" .
echo preseed.cfg | cpio -o -H newc -A -F install.amd/initrd
rm preseed.cfg
gzip install.amd/initrd

find -follow -type f -print0 | xargs --null md5sum >md5sum.txt

# Extract MBR template file to disk
dd if="$OLD_PWD/$ISO" bs=1 count=432 of="isohdpfx.bin"

xorriso -as mkisofs -o "$NEW_ISO" -isohybrid-mbr "isohdpfx.bin" \
  -c isolinux/boot.cat -b isolinux/isolinux.bin -no-emul-boot -boot-load-size 4 \
  -boot-info-table -eltorito-alt-boot -e boot/grub/efi.img -no-emul-boot \
  -isohybrid-gpt-basdat "$WD"

cd "$OLD_PWD"

cp -f "$WD/$NEW_ISO" .

rm -r "$WD"
```

One caveat, I want to have prod and dev environments which are going to have different configs. So I created a python script that takes preseed.cfg.j2 file, templates it using jinja2 and outputs the preseed.cfg file for my .iso:

```python

#!/usr/bin/env python

import os
import sys
import argparse
from jinja2 import Environment, FileSystemLoader, exceptions


def render_template(template_file, output_file, context):
    """
    Renders a Jinja2 template with a given context and writes it to an output file.
    """
    try:
        # Get the directory containing the template file
        # This allows Jinja2 to find the template correctly
        template_dir = os.path.dirname(os.path.abspath(template_file))
        template_name = os.path.basename(template_file)

        # Set up the Jinja2 environment
        # The FileSystemLoader loads templates from the file system
        env = Environment(loader=FileSystemLoader(template_dir))

        # Load the template
        template = env.get_template(template_name)

        # Render the template with the provided context
        rendered_content = template.render(context)

        # Write the rendered content to the output file
        with open(output_file, "w") as f:
            f.write(rendered_content)

        print(f"Successfully rendered template '{template_file}' to '{output_file}'")

    except exceptions.TemplateNotFound:
        print(f"Error: Template not found at '{template_file}'", file=sys.stderr)
        sys.exit(1)
    except Exception as e:
        print(f"An unexpected error occurred: {e}", file=sys.stderr)
        sys.exit(1)


def main():
    """
    Main function to parse arguments and initiate template rendering.
    """
    parser = argparse.ArgumentParser(
        description="Render a Jinja2 template using environment variables."
    )

    # Define command-line arguments
    parser.add_argument("template_file", help="Path to the input Jinja2 template file.")
    parser.add_argument("output_file", help="Path for the output rendered file.")

    args = parser.parse_args()

    # Use all environment variables as the context
    # We pass `dict(os.environ)` to create a simple dictionary
    # that Jinja2 can easily work with.
    context = dict(os.environ)

    render_template(args.template_file, args.output_file, context)


if __name__ == "__main__":
    main()
```

The above is vibe coded because I didn't feel like spending much time on some simple jinja2 template script.

One last thing, I don't want to download the original Debian .iso manually and then have to run 2 scripts. So here's the super simple script that will run the render.py and the mkpreseediso.sh scripts (the .iso must be downloaded manually because it only needs to be downloaded only once):

```shell

#!/bin/bash

set -e
OLD_ISO="$1"
NEW_ISO="$2"

python render.py preseed.cfg.j2 preseed.cfg
./mkpreseediso.sh "$OLD_ISO" "$NEW_ISO"
```

Now, all of the environment variables used in the template file need to be written to a file that will be imported during script execution. Next, the script can be run - `bash -c 'set -a; source fearless-local.env.gitkeep; set +a; ./main.sh debian-13.1.0-amd64-netinst.iso fearless-local.iso'`

I don't guarantee that the above files won't change. Up to date and best effort docs will be available in my git repository.

Now, Debian can be installed. If you only have a single disk then selecting the standard 'Install' should make the installer go through the install process without any more clicking required. Neat.

After the reboot a temporary passphrase will be needed to be provided. This one I am not bothered with. Because if I make a typo, I will be able to just try again. If I did this during installation, then I would have to go through the whole installation process once again. I guess that makes sense.

Anyway, Debian should be installed now and the host should be up and running. We can now proceed to setting up the Tang server.

### Tang server setup

Tang was initially created by some clever guys at RedHat so since there were no 'official' Docker images for me to use, I decided to use the RedHat 10 Tang Docker image from the 'subscription' repository. This means that I am at the mercy of the big red fedora but I don't care. I have Tang keys backed up and if the need arises, I will be able to switch to a custom Docker container. Maybe running AlmaLinux or something.

I already had the following in the readme for tang in my repo so I figured I'll just paste it here because it's self-explanatory:

1. Log in to RedHat registry
2. Run the tang container
  prod - `podman run -d -p 7500:8080 -v tang-keys:/var/db/tang --name tang registry.redhat.io/rhel10/tang`
3. Generate tang.yml file
  `podman kube generate tang -f tang.yml`
  `podman kube generate tang-dev -f tang-dev.yml`
4. Kill and remove the container
  `podman kill tang && podman container rm tang`
  `podman kill tang-dev && podman container rm tang-dev`
5. Run `podman kube play tang.yml`

More on standard operations when using Tang in some later sections.

### Host configuration, Clevis-Tang binding and emergency passphrase

The next thing is to run an Ansible playbook. I am too lazy to copy paste this stuff here so if you want to take a look at its contents then go through git. It should have been shared at the top of the article

Anyway, once this is done, Proxmox should be installed, Clevis should be bound on the last keyslot to the Tang server, emergency passphrase should be in the first keyslot and the temporary bootstrap passphrase should have been deleted.

Now, Tang can be stopped - `podman kube down tang.yml`

### Standard operation

When at home and I need to reboot my server, I can unlock my laptop, run the Tang server and boot up the server and then stop it:

1. Run `podman kube play tang.yml`
2. Boot up the host running Clevis
3. Profit
4. Stop tang `podman kube down tang.yml`

Volumes are persistent so this should work for quite some time. Now, volumes holding the keys should be backed up. Well, exported, then encrypted and then backed up. But I have not set this up yet. I want to focus on getting my services up and running on Talos Linux.

### Emergency operation

In case my laptop dies before I set up the keys backup solution, the emergency passphrase can be used for, as the name implies, emergency operations.

Simply connect a display to the server and boot it up. Then, the emergency passphrase can be input when in the initramfs

### Conclusion

Thanks to some questionable protocol decisions at SideroLabs I have to go back to using Proxmox because of reasons. I hate this approach and I will try to set this thing up natively in Talos but today is not the day. Next, I'll set up Talos as a VM and passthrough the HBA directly to it and make a one box for everything kind of thing
