---
title: 'Multi-node homelab setup with Proxmox'
description: "Part 1."
date: '2026-10-29T22:26:41+01:00'
tags: ["homelab","proxmox","virtualization","cluster","cert"]
author: "rudfrank"
draft: false
showToc: true
series: ["Homelab"]
---

## hardware

I want a multi-node setup to test clustering functions.

**Node1 (pve)** will be always on. It is an energy efficient minipc ([BMAX B7A Pro](https://www.bmaxit.com/MaxMini-B7A-Pro-pd565690948.html) with Ryzen 5 and maxed out RAM) I bought used for this purpose. I will pin important services to this.

**Node2 (pve2)** is a repurposed 15-years-old Intel X97 desktop, upgraded with a pcie-nvme adapter module with a 256GB nvme ssd (leftover from a laptop upgrade), as well as a sata ssd for storage, and a used Nvidia 3060 12GB (I want to experiment with light local LLMs). \
It will be powered on as needed, as it is quite power-hungry.

## install
Download the newest (9.2 as of writing) Proxmox VE installer iso from https://proxmox.com/en/downloads/proxmox-virtual-environment/iso
Burn to usb: `dd bs=1M conv=fdatasync if=./proxmox-ve_9.2-1.iso of=/dev/sdX` (find the correct device with `lsblk`)
Then boot from the usb and install both nodes. Simple 'next-next-finish' process, no need to elaborate.

For hostnames I simply choose *pve* and *pve2*.
I use a subdomain of my public domain for the homelab, it makes thing (like https) easier.
I select IPs so that I can easily manage them in my firewall (eg. consecutive IPs in a /31 cidr-range).

The hostname entries should not be in the public dns! My mikrotik router will be the dns and dhcp server in the network.

### complications with Node2
After install I noticed that it cannot boot from the nvme ssd in the pcie-adapter (of course).

Even after reinstalled to the sata ssd, it looks like that the installer wrote grub to the nvme (as it sees it as first device), so no boot for me... I need to reinstall grub:
1. I *rescue-boot* from the installer to the freshly installed system, login as root
2. using `lsblk` I identify the ssd -- it is */dev/sdb* for me -- it is easy to find, as you see the `pve-...` lvm volumes under it.
3. reinstall grub to the sata ssd: `grub-install /dev/sdb` \
First I've got an error: `grub-install: error: cannot find EFI directory.` \
Let's mount the efi directory: `mount /dev/sdb2 /mnt` \ 
Now I can install grub: `grub-install /dev/sdb --efi-directory=/mnt/efi`
4. update the grub2 menu: `update-grub`
5. reboot

After reboot I can login as root via ssh using the IP I choose during install and continue the setup.

## initial setup
These steps are done on both nodes.

### upgrade
The official pve image is based on debian, so `apt` can be used.

#### setup free repos
For the homelab I won't have enterprise license, so I need to add the free sources to be able to use `apt udate` and `apt upgrade` \
Let's create ` /etc/apt/sources.list.d/proxmox.sources` file with content:
```
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
```
Disable the pve-enterprise sources: \
`echo "Enabled: no" >> /etc/apt/sources.list.d/pve-enterprise.sources`

Add ceph-free to `/etc/apt/sources.list.d/ceph-free.sources`:
```
Types: deb
URIs: http://download.proxmox.com/debian/ceph-tentacle
Suites: trixie
Components: no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
```
Disable the enterprise ceph sources: \
`echo "Enabled: no" >> /etc/apt/sources.list.d/ceph.sources`

#### apt
Now we do a classic `apt update && apt upgrade`
Probably finds many packages to upgrade, continue and wait.
When finished, `reboot` then cleanup: `apt autoremove && apt autoclean`

#### upgrade firmwares
Install firmware update manager: `apt install fwupd`
Refresh the cache: `fwupdmgr refresh`
Upgrade: `fwupdmgr upgrade`, reboot if needed.

### web setup
Login to the webui in browser: `https://[node-ip]:8006`
In the *Server View* select the current node under *Datacenter*.

#### import certificate
I use an internal self-hosted [step-ca](https://smallstep.com/docs/step-ca/) to generate certs, so I need to import the CA as well. \
For you it might not be relevant; you can use purchased ssl certs, or the built-in [Let's encrypt](https://pve.proxmox.com/wiki/Certificate_Management#sysadmin_certs_get_trusted_acme_cert) (but only with DNS-challenge, as for the HTTP, port 80 of the node would be need to be reachable from the internet -- I want to avoid it).
1. I`scp` my root ca to the node, then copy it into ` /usr/local/share/ca-certificates/`
2. `ssh` to the node, then issue `update-ca-certs`, then reboot.
3. on the web UI select the node, then *System -> Certificates*, select *Add* under *ACME*
    - Challenge Type: HTTP
    - Domain: use the full fqdn -- the hostname with the lab domain.
4. *Add ACME Account*
    - Account Name: stepca
    - E-mail: fill it...
    - ACME Directory: Custom (you might need Let's Encrypt V2)
    - URL: https://[stepca-hostname]:443/acme/acme/directory
5. Then *Order Cert* -- if all went well cert is generated and installed.

Note: on the *stepca* node an ACME provisioner was needed to be set up first (default is only JWK) before I could add it to proxmox: \
`step ca provisioner add acme --type ACME --ca-url https://[stepca-hostname]:443  --root /etc/step-ca/certs/root_ca.crt --ca-config /etc/step-ca/config/ca.json`

From now on, I can reach the node with its domain without https warnings (given that the CA is installed to all the nodes used to manage them).

#### create the cluster
1. On the web ui on node1 (pve), under *Datacenter -> Cluster*, *Create Cluster*. I simply name it homelab.
2. Select *Join Information*, then *Copy Information*
3. Open web ui on node2 (pve2). Under *Datacenter -> Cluster*, *Join Cluster*. Paste the join information from node 1.
4. All fields will be filled, except your password. Enter it, then *Join*.

Note: If you setup TFA/MFA to your proxmox user, you have to use the cli to join instead. On node 2 via ssh: \
`pvecm add [node1-fqdn] -link0 [node2-ip] --use_ssh`, eg. `pvecm add pve.lab.mydomain.com -link0 192.168.0.22 --use_ssh`

---
We have our cluster ready. From now on, I can see and manage *node 2 (pve2)* via the *node 1 (pve)* web ui, under *Datacenter*.

I setup my disks and storages via the web ui and define my resource pools and add users and permissions. At minimum, I define a vmmanager group and a normal user paired with it. 

In the next part we will start to use the cluster and deploy vms and some lightweight lxc containers.

