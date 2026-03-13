---
title: Private Server Build
params:
  images:
    - /media/projects/private_server_build/cover-photo.jpg
---
> I wanted to stop paying for services I could run myself. 
> What started as a simple media server turned into a deep dive into self-hosted infrastructure — Proxmox, containerization, reverse proxying, and Zero Trust security, all running on a $200 eBay machine. 
> No monthly fees. Full control.

## Hardware & Operating Systems

Considering the steep increase in hardware components, I launched this project with an old Lenovo ThinkCentre (10ST001XUS M720s) due to their reliability in enterprise environments (sourced from eBay).

![Internal Snapshot](/media/projects/private_server_build/ebay-images.jpg)

### Hardware

**Chassis:** This is where I made my first mistake - this model is specifically Small Form Factor (SFF), which really limits the internal space for future upgrades.
On top of that, low profile PCIe devices generally cost a bit more and adding something like a dedicated graphics card could introduce cooling issues. 

**Central Processing Unit (CPU):** Machine comes from the factory with an Intel i5-8400 - 6 cores, 2.8 GHz base clock, 8th generation.
The primary benefit of this CPU is the Intel QuickSync hardware transcoder, that allows me to transcode multiple 4k movies without the need for a graphics card (which this model lacks).
Overall, it's not the latest chip design (nor the fastest), but it'll handle my use-cases just fine.

**Random Access Memory (RAM):** Came with a single 8GB Ramaxel card, which was not nearly enough to run multiple services. 
I installed a 16GB Ripsaw, and a generic brand 8GB stick I pulled from another machine destined for the junkyard; 24GB of memory total.
All sticks are DDR4, which serves my purpose just fine, the system will simply downclock to the slowest card.

![Internal Snapshot](/media/projects/private_server_build/internals.jpg)

**Storage (Boot):** Apparently this model comes with an open M2 slot from the factory, you simply need to order an aftermarket caddy to stabilize the drive post-install.
I had a 512GB SSD from another build, and I even went the extra mile and ordered a custom heatsink to help extend lifespan.
ThinkCentre's unfortunately boot and store media on the same device, which opens the door to massive data loss (boot drives get serious mileage, can fail faster) - plus the SSD upgrade will make the machine run much faster.

**Storage (Media):** Came stock with a 3.5" 1TB Western Digital HDD, but I upgraded to a single 8TB Seagate IronWolf. 
I would've mirrored with an additional equal-size drive, but I neglected to confirm the size of the 2nd drive slot. 
Apparently this machine ships with a 2.5" backup slot, and that size tops out around 5TBs while being significantly more expensive, overall.

![Exterior Snapshot](/media/projects/private_server_build/external.jpg)

**Storage (Backup):** To solve the mirroring issue, I decided to order an 8TB External Desktop HDD. My primary concern isn't really the container images, it's the media files.
The backup time is quite significant (USB 3.0), but I plan to implement incremental backups to help mitigate.
I'd rather deal with slow backups than risk losing terabytes of movies/shows, or limiting my HDD capacity to whatever I can find in 2.5" sizes. 

**Network Interface:** Comes stock with an Intel 1219-V which can hit Gigabit speeds with relatively low CPU overhead. Excellent Linux driver support.
Only downside is you have a single RJ45, so I'm limited to just direct gateway connection. 


### Operating Systems

**Proxmox VE** is the bare-metal hypervisor running directly on the hardware — think of it as the foundation that everything else is built on top of.
It gives me a clean web interface to manage all of my containers and virtual machines from one place, and it's completely free and open source.
It's built on Debian under the hood, so there's nothing exotic going on at the OS level.

**Ubuntu** is my choice for LXC containers. It's lightweight, stable, and has massive community support — if I run into an issue, someone has already solved it.
Unprivileged LXC and Ubuntu is a well-documented combination, which matters when you're learning.

**Ubuntu Server** is what I run inside my Virtual Machines. It ships without a desktop environment, which means lower resource overhead on headless machines I'm only ever accessing via SSH.
No point allocating RAM to a GUI nobody is looking at.

**Docker** runs inside the Ubuntu Server VMs, managing multi-container stacks via Compose files.
It's worth noting that I made a deliberate decision to run Docker inside VMs rather than LXCs — Docker inside a privileged LXC introduces security tradeoffs I wanted to avoid.
A dedicated VM keeps the Docker workloads isolated without compromising the security model of the rest of the infrastructure.

## Functionality

What started as a home media server has grown into a multi-service homelab running across six isolated containers and virtual machines.

![Proxmox Dashboard Screenshot](/media/projects/private_server_build/proxmox-dash.png)

| ID               | Type | OS            | Application(s)                        | Purpose                                                                                                                             |
|------------------|------|---------------|---------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| 100 'media'      | LXC  | Ubuntu        | n/a                                   | Dedicated storage for media files                                                                                                   |
| 101 'jellyfin'   | LXC  | Ubuntu        | Jellyfin                              | Reads media storage and streams movies to the device, hardware transcodes if need be                                                |
| 102 'media-apps' | VM   | Ubuntu Server | Docker: Gluetun, + various media apps | Hosts Docker-managed media automation services, routed through an outbound Gluetun VPN for network isolation                        |
| 103 'homepage'   | LXC  | Ubuntu        | Homepage                              | Provides centralized monitoring/management of all apps and services                                                                 |
| 104 'nginx'      | VM   | Ubuntu Server | Docker: Cloudflare DDNS, Nginx        | Hosts Nginx reverse proxy, and custom container to auto update my DNS records                                                       |
| 105 'ai-stack'   | VM   | Ubuntu Server | Docker: Ollama, Web-UI, N8N           | Currently housing my automation and AI applications. I've been testing self-hosted AI options (Qwen) to integrate into my household |

### Other Notable Services

**Samba File Share:** As my media storage grows, it will become increasingly difficult to manage file storage just by accessing the console. 
I decided to implement an SMB Share so I could manage my collections from my main computer, leveraging the File Explorer as my GUI.

![Samba Share Screenshot](/media/projects/private_server_build/network-fileshare.png)

**SSH Access:** Accessing the console via Proxmox Web Login has its quirks, so I decided to install OpenSSH and import my keys. 
I implemented a passphrase for additional protection, and I disabled root user login on all machines/containers to prevent direct root access via SSH.
In addition to that, I figured as services expand - it'll be pretty hard to track IP addresses across containers and machines - so I wrote a quick config doc that way I can connect via keyword instead of looking up the Ipv4.

![SSH Passphrase Login](/media/projects/private_server_build/ssh-login.png)

![Snapshot of ssh Config Doc](/media/projects/private_server_build/config-ssh.png)

**Cloudflare Zero Trust:** While planning out the topology of the system, I decided to implement a zero trust tunnel for all admin webpages. 
Applications like the Proxmox Web Login, or the Nginx admin site shouldn't be exposed to the public since I am the only individual who should be accessing them, regardless of how this system evolves. 
The zero trust tunnel works by installing `cloudflared` on your machine to maintain a persistent connection to Cloudflare, this means no inbound ports are required on your router. 
When you attempt to access one of my subdomains, your request first goes to the Cloudflare edge, and the device's identity is checked against the access policy that I've configured.
Once validated, `cloudflared` routes directly to the internal IP:PORT associated.

![Zero Trust Auth Prompt](/media/projects/private_server_build/cloudflare-zerotrust.png)

Benefits:
- Home IP is never exposed, traffic never even touches it
- No open ports on my router
- Authentication happens before connection is permitted
- Custom firewall rules I can configure to my liking


## Virtual Topology

I designed the topology to balance application access with general security practices.

![Virtual Topology](/media/projects/private_server_build/net-diagramv2.png)

### Potential Attack Vectors
- **Physical Access:** This is my highest security risk, I do not currently have a locked server rack - so if someone were to access my home, they could potentially have unrestricted access to all of my equipment(assuming they know what they're doing).
I'm following basic hardening practices to prevent direct access (separating root & user accounts, least privilege, high entropy passwords, clipboard timers, etc), but the ultimate solution will be a locked rack.
- **Wireless Access Point:** An attacker could connect to my services if they've gained access to my WAP. To combat this I've implemented WPA3, with a randomized 35 character password to harden against direct WiFi cracking.
In my next project, I'll be segmenting my network to isolate high-risk IoT devices, and potentially implementing a DMZ to keep the internet-accessible services away from high-trust devices.
I also have plans for implementing 802.1X authentication, alongside IDS/IPS. 
- **Nginx Reverse Proxy:** Jellyfin is exposed via the nginx reverse proxy, opening the door to a potential attack vector if Jellyfin is somehow compromised. 
My best bet (for now) is to keep Jellyfin up to date, implementing strict firewall rules (nginx), and continuing with my plan for a screened subnet.
I've implemented MFA and a strong 35 character randomized password to help combat brute forcing.
- **Cloudflare Zero Trust Tunnel:** This access path is the least likely to cause any issues. Cloudflare Zero Trust validates the requesting client by requesting a code from my encrypted email (locked behind MFA + High Entropy Passkey), before connecting the client to the service.
Additionally, when the client attempts connection, they meet the Cloudflare IP address - not mine. You have zero connection to any of my services until authenticated. I'm using this for high-risk applications like admin panels.
I would include Jellyfin in this tunnel, but SmartTV applications don't have the ability to authenticate the Cloudflare tunnel, the server connection simply fails.


## Future Plans

**General Container Consolidation:** As I was moving through this project, I was still learning the ins and outs of segmentation.
At this point I feel some of my containers/machines could be condensed which would help me utilize my resources more efficiently, eliminating some VM overhead.
This will require me to migrate services and do a bit of backtracking, so it's at the end of my list.

**Dedicated AI Node:** After some initial rounds of testing with Ollama/Qwen3.5:9b - I need a dedicated graphics card for better performance.
Initially I figured I could find a low profile GPU, but I think the smarter and more cost-effective strategy is to run an additional node (full-sized this time) to prevent overheating.

**Refine Security Controls:** At this point most of my ACLs are vague/weak, so they could use a bit of tweaking. 
I'd also like to migrate Jellyfin to a screened subnet in order to isolate any future exploits. 
In addition to that, I'm considering running a wireguard node that can be connected to remotely, which would then prevent me from needing to open 443/tcp to access Jellyfin.
Wireguard would just forward me to Jellyfin.

**NAS Storage:** Having a single SATA drive, and a cron backup running once per week isn't exactly my ideal setup. 
I'd prefer to segment media storage out of the node completely. 
I've looked at a few pre-built options with up to 8-10 bays, but they can be pretty expensive and I need to investigate how I can build my own first. 
I prefer the DIY route so I can "learn while I play" so to speak. 



The homelab is never really finished — it just evolves. What started as a media server is now a legitimate self-hosted infrastructure stack, and every problem I ran into taught me something I couldn't have learned in a classroom. The next phase is already in motion.