---
title: Building a Private Media Server
---

> This writeup focuses primarily on the hardware side of my first private server build.
> The goal of this project is to have an efficient Proxmox server, running multiple VM's, providing me with various, dedicated home services.


## Choosing a Base Model

The goal for this project is to build a versatile private server, but I want to stick to a basic budget of <$500. 

I found this old Lenovo Thinkcentre on eBay for ~$130. This should serve as a great base to build upon, considering the budget.

![Lenovo eBay Listing](/media/PrivateServerBuild/lenovo-ebay-listing.jpg)


Per the listing it comes with:
- Intel Core i5 Processor (8th gen) w/ Quick Sync)
- 8GB DDR4 RAM (+ 3 Additional Slots)
- 1TB 3.5" HDD Storage (Addtl 2.5" slot for future expansion)

The reasons I opted for this model:
- 8th Gen Processor + Quick Sync creates less overhead for media transcoding (Project #1 is Media Streaming)
- ThinkCentre models are commercial lines, they're built to last
- Open m.2 Slot for Upgraded Boot Drive (SSD)

For $130 bucks, this is a great starting point. 


## Hardware Upgrades

First thing on the docket - install the dedicated SSD boot drive for better performance. 

I checked Facebook Marketplace, found a guy selling SSDs in various sizes for incredible cheap (I paid $15 for 512gb) and he even provided a CrystalDisk report showing near zero hours. 

![SSD Expansion](/media/PrivateServerBuild/ssd-expansion.jpg)

I ended up needing an aftermarket caddy to secure the drive ($12), and I opted for a custom heatsink as well ($15).

I'm a bit nervous of the heat since this model is so compact, but the heatsink should help. 

Next, I swapped in new memory - specifically a pair of Ripsaws I bought off Facebook Marketplace ($20 STEAL)

![RAM Swap](/media/PrivateServerBuild/ram-swap.jpg)

One of the sticks ended up being DOA, but for $20 - 16GB of DDR4 isn't something I'll complain about.
Especially since I could just downclock the Ripsaw and use the 8GB Ramaxel card that came with the Lenovo.

I also took a second to clean off the old, crusty thermal paste and applied it fresh to help keep core temperature's stable.

![Fresh Thermal Paste](/media/PrivateServerBuild/thermal-paste.jpg)

And finally, I swapped the current Western Digital 1TB HDD for a Seagate IronWolf 8TB NAS.

![Fresh Thermal Paste](/media/PrivateServerBuild/hdd-expansion.jpg)

Now for storage backup, I would have preferred to do a ZFS pool, but I overlooked one small factor: the additional HDD slot is 2.5", not 3.5". 

Fat chance I'll be able to find 8TB HDD in 2.5" - so I went with an external unit, and I'll just mount it manually at the shell. 


# Installing & Configuring the Operating System

Since the goal of this server is to be multi-functional, I landed on Proxmox for my operating system. 

![Proxmox Website](/media/PrivateServerBuild/proxmox-web.jpg)

Proxmox is an open-source virtual environment, tailored for managing virtual machines and containers. 
This should give me the flexibility to expand into future projects like managing my network with OPNsense, or creating a media library with Ubunto.

I downloaded the Proxmox ISO file, and created a bootable drive with Rufus. 

![Creating a Bootable Drive](/media/PrivateServerBuild/rufus-boot.jpg)

Once the drive had been successfully flashed, I moved to my server - fired her up and smashed F12 to get into NetBIOS.
Then I let it do it's thing.

Once it was done, I set my credentials and assigned a static IP address outside my network's DHCP range.

and voila - she's alive. 

![Proxmox Dashboard](/media/PrivateServerBuild/proxmox-dashboard.jpg)

By default, Proxmox assumes enterprise use, so we'll need to configure it to the No-Subscription plan via a convenient post-install script.  

![Post Install Script](/media/PrivateServerBuild/post-install.jpg)

Here's the [link](https://community-scripts.github.io/ProxmoxVE/scripts?id=post-pve-install) if you're interested.

And once that was complete, I ran `apt update` to check for updates - and to no surprise there were quite a few - so I upgraded.

Last change we need to make, enabling IOMMU (Input–Output Memory Management Unit).

IOMMU essentially provides direct access for VMs to safely use physical hardware - and since one of the usecases for my server is to build a streaming app, I'll want that VM to have access to the Intel Quick Sync function (media transcoder).

In shell, type: `nano /etc/default/grub` which should bring you here:

![IOMMU Update](/media/PrivateServerBuild/updated-iommu.jpg)

Update the line `GRUB_CMDLINE_LINUX_DEFAULT="quiet"` to `GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"`

Hit Ctrl + O (saves) > Enter (confirm save) > Ctrl + X (exit editor)

Now run `update-grub`

then `reboot`

After reboot, you can check to ensure IOMMU is enabled by using `dmesg | grep -i iommu`

![IOMMU Enabled](/media/PrivateServerBuild/IOMMU-enabled.jpg)






