---
title: Self-Hosted Media Streaming Infrastructure
---
## Overview

You can ask anyone who knows me - I'm *definitely* a movie guy.

Now the problem is, I have a **huge** collection of 4k Ultra-HD movies - but they're locally stored at home on my desktop. 

This means:
- I can't access them while traveling
- I can't share them with family or friends
- If I want to take advantage of my 4k OLED TV, I have to Multi-cast (not user-friendly, poor performance)

So I'd say it's about time I build my very own, self-hosted streaming app that can be accessed anywhere in the world - so long as I have internet access.

Yes - I *could* buy a ready-made server, but where's the fun in that?

## Hardware & OS

First things first, we need some hardware. I love my desktop - it's a pre-built gaming PC and definitely has the juice to stream high quality movies, but I don't exactly want to burn up my hardware since servers are always on. 

So I headed to eBay and picked up an old Lenovo to use as my base machine. 

![Lenovo ThinkCentre M720s - eBay Listing](/media/MediaServerBuild/lenovo-ebay-listing.jpg "Lenovo ThinkCentre M720s Intel i5 8th Gen w/ Quick Sync")

ThinkCentre models are business-grade, so they're built to last. On top of that, this model comes with 'Intel Quick Sync' hardware transcoding. I checked the specs, and this should allow me to bypass the need for a dedicated graphics card.

Once it arrived, I made a few upgrades:
1. Installed 512GB SSD for my dedicated boot drive
2. Replaced the 1TB Western Digital HDD with an 8TB IronWolf HDD.
3. Added a 16GB Ripsaw DDR4 (Now has 24GB of RAM Total)
3. Gave the CPU fresh thermal paste (who knows how old it was?)

Now, I definitely made a mistake here. From my research, I was under the impression that this machine had 2 - 3.5" HDD cages, but when it arrived I found that it *actually* had one 3.5" cage, and one 2.5" cage. 
So that pretty much killed my dreams of mirroring my hard drives for redundancy, but we're going to make it work by using rsync to regularly backup our disk images, as well as our media storage.

Given that, I ordered this 8TB External HDD. 

![Seagate Expansion Desktop Drive](/media/MediaServerBuild/HDD-external-ebay.jpg "Seagate Expansion Desktop Drive 8TB USB 3.0" )

For the operating system - I decided to go with [Proxmox VE](https://proxmox.com/en/products/proxmox-virtual-environment/overview). 

Proxmox is an enterprise-grade, open source server management platform which will allow me to run as many Linux Containers/Virtual Machines as my hardware will allow.

It was super easy to install - I simply grabbed the [latest ISO for Proxmox VE](https://proxmox.com/en/downloads), and flashed it to a spare USB with [Rufus](https://rufus.ie/en/). 

From there, I just followed the installation process, and about 10 minutes later I was logged into my Proxmox Web client. 

![Proxmox Dashboard](/media/MediaServerBuild/proxmox-dashboard.jpg "Quick Snapshot of Proxmox VE Dashboard" )

*ignore the '16GB of RAM' showing, I originally swapped the factory installed 8GB for 16GB but then realized that was stupid - just run both 🤷🏼‍♂️*

## Virtualization Setup

To help keep things more secure, I decided to store my media in its own LXC, and operate Jellyfin out of another. (both LXC's unprivileged, of course)
I also wanted to host a few support/automation apps, and I'll be doing that with Docker via a Virtual Machine running Ubuntu Server.

| Name                | Type | Cores   | RAM  | Disk  | OS            |
|---------------------|------|---------|------|-------|---------------|
| Media  (101)        | LXC  | 2 Cores | 2GB  | 64GB  | Ubuntu        |
| Jellyfin  (102)     | LXC  | 4 Cores | 3GB  | 32GB  | Ubuntu        |
| media-apps    (103) | VM   | 4 Cores | 12GB | 64GB  | Ubuntu Server |

After I launched all of my containers/machines - I logged into the node shell and ran a quick `sudo apt install openssh`, then added my public key so I could easily log in via my host pc. (Using the console inside Proxmox is a PAIN)

![OpenSSH](/media/MediaServerBuild/openssh-console.jpg "Yes you can see my IP but does it really matter?" )

Technically, I could install OpenSSH on each container & the VM - but you can easily manage the entire datacenter from the node - it's all preference.

## Media Storage & Streaming

So in order for Jellyfin and the Media storage to work correctly, we need to:
- Activate Intel QuickSync Hardware Transcoding (/dev/dri/renderD128)
- Mount the 'tank' (Internal HDD) to both LXC's, but enforcing 'read only' least privilege on Jellyfin

Here's a quick snapshot of the configurations for both containers:

```bash {filename="/etc/pve/lxc/100.conf"}
root@pve:/# cat /etc/pve/lxc/100.conf
#ipv4 = 192.168.1.3/24
arch: amd64
cores: 2
features: nesting=1
hostname: media
memory: 2048
mp0: tank:subvol-100-disk-0,mp=/data,size=6700G
mp1: local-lvm:vm-100-disk-1,mp=/docker,size=32G
net0: name=eth0,bridge=vmbr0,firewall=1,gw=192.168.1.1,hwaddr=BC:24:11:D3:BA:02,ip=192.168.1.3/24,type=veth
onboot: 1
ostype: ubuntu
rootfs: local-lvm:vm-100-disk-0,size=64G
swap: 1028
unprivileged: 1
root@pve:/#
```

```bash {filename="/etc/pve/lxc/101.conf"}
root@pve:/# cat /etc/pve/lxc/101.conf
#ipv4 = 192.168.1.4/24
arch: amd64
cores: 4
features: nesting=1
hostname: jellyfin
memory: 3072
mp0: tank:subvol-100-disk-0,mp=/mnt/media,ro=0
net0: name=eth0,bridge=vmbr0,firewall=1,gw=192.168.1.1,hwaddr=BC:24:11:C0:6A:D8,ip=192.168.1.4/24,type=veth
onboot: 1
ostype: ubuntu
rootfs: local-lvm:vm-101-disk-0,size=32G
swap: 1028
unprivileged: 1
lxc.idmap: u 0 100000 65536
lxc.idmap: g 0 100000 108
lxc.idmap: g 108 993 1
lxc.idmap: g 109 100109 65427
lxc.apparmor.profile: unconfined
lxc.cap.drop:
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir 0 0
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file 0 0
root@pve:/#
```

You may notice the Jellyfin config file is a bit longer, and that's because I initially launched LXC 101 as a *privileged* container, and then I had to go back and change it, and update the privilege translations.
This was a major error on my part, and I paid the price by having to educate myself on privilege translations. 

So at this point, LXC 100 'Media' stores files, and LXC 101 'Jellyfin' can read and stream them using Intel QuickSync.

{{< callout >}}
Hardware transcoding is **crucial** because this machine has no graphics card *and* is a small form factor design - meaning I have limited options for GPUs to begin with. 
From my research, Intel QuickSync (8th Gen or later) can technically transcode up to 3 different 4k streams at a time, so our use case should fall right in line with this device's capabilities.
{{< /callout >}}

And finally, we just need to install Jellyfin on LXC 101, and make some quick setting adjustments inside the app. 

I went ahead and used this command straight from Jellyfin's [Documentation](https://jellyfin.org/docs/):

> `curl https://repo.jellyfin.org/install-debuntu.sh | sudo bash`

Then I simply logged into Jellyfin, created a username & password, and headed straight for the settings to enable hardware transcoding.

![Jellfin Settings Pt.1](/media/MediaServerBuild/jellyfin-transcoding.jpg "Navigate to Playback > Transcoding" )

From here, I selected my iGPU brand, and specified the path to the device. I also deselected every encoding type except HEVC, H264, and HEVC 10bit - DYOR.

Additionally, if you have movies with HDR (tone mapping), you'll want to check these boxes to avoid blowing out your CPU.


![Jellfin Settings Pt.1](/media/MediaServerBuild/jellyfin-transcoding2.jpg "Most newer movies include HDR" )


## Automated Download Pipeline

Moving on to the VM running Ubuntu Server. First thing I did (of course after running
`sudo apt update && sudo apt upgrade`) was to install Docker.

And to our luck, Docker **also** has a beautifully easy 2-step process for install,
according to their [documentation](https://docs.docker.com/engine/install/ubuntu/).

From there, I deployed a handful of containerized services via Docker Compose —
including a VPN client (Gluetun) to route all download traffic through an encrypted
tunnel, a download client, and a few API-based media management services that
automatically monitor, fetch, and organize new media into the correct library folders.

All download traffic is routed exclusively through the VPN container — if the VPN
drops, traffic stops. No leaks.

Once media is downloaded and organized, it lands in the shared storage volume where
Jellyfin picks it up automatically — no manual intervention required.

## Networking

I elected to go with static ip assignments to help keep the virtualization organized. 
My LAN is a singular subnet (192.168.1.0/24) and my ISP's gateway comes with a default DHCP scope of 192.168.1.100 thru 192.168.1.200 - 
so matching the container ID's with IP Addresses wasn't going to happen.

| Name       | ID   | IP Address  |
|------------|------|-------------|
| pve        | Node | 192.168.1.2 |
| media      | 101  | 192.168.1.3 |
| jellyfin   | 102  | 192.168.1.4 |
| media-apps | 103  | 192.168.1.5 |

DNS: I went with CloudFlare: `1.1.1.1, 1.0.0.1`
Ethernet: Cat6a

On top of that I decided to `apt install samba` because my movie collection is quite large, and dealing with the console *every* time I want to manage my library, won't be very efficient.

![Windows Network Share](/media/MediaServerBuild/net-share.jpg "Sharing over SMB provides quicker access to my library" )

Still working on setting up nginx for reverse proxy, and purchasing a domain for my TLS certificate. Will update here soon or maybe I'll add a 'Hardening' section.

## Reflecting on Challenges & Lessons

### Troubleshooting Linux Permissions

This was my first major project that involved linux commands (aside from basics like `ping` & `nmap`), so I essentially speed-ran an introduction to Linux. 
Most notably, I ran into major issues with my file/directory permissions. I had zero clue how to read something like `drwx---r-x` - which led me down a frustrating loop of Jellyfin being unable to read the device driver (`/dev/dri/renderD128`), and the error code was quite ambiguous. 
I was able to pinpoint the permission issue with a little help from Claude, and monitoring `journalctl`. After a quick education session with Claude, and a few YouTube videos - I fixed the permission issue. 

### iGPU Passthrough Failure

Initially, I had Jellyfin hosted through docker (separate compose.yaml file) on the same VM as my automation & organization clients - but passing through the hardware transcoder was proving to be quite difficult. 
I kept getting one error after another, and most of them we're very ambiguous. 
I confirmed the permissions, added the device to path, *specified* the path in Jellyfin, made sure the device was correctly added to the VM config file - but it just would **not** cooperate.
After a few days of research, I ended up finding that Jellyfin actually recommends for you to use a container instead when hardware transcoding. 

So I deleted the docker container running Jellyfin, and installed it directly into an LXC - no more issues.

You can verify hardware transcoding is active by seeing little to no CPU usage, I used `htop` but you can also use `intel_gpu_top` (just make sure to install both)




