---
title: I Built My Own Netflix From Scratch
params:
  images:
    - /media/MediaServerBuild/cover-photo.jpg
---
> I have a massive 4K movie collection sitting on a local drive — unwatchable away from home, unshareable, and stuck behind a bad multicast setup. 
So I built my own streaming server from scratch: Proxmox, Jellyfin, automated downloads, hardware transcoding, and a hardened public setup with Nginx and Cloudflare Zero Trust. 
No monthly fees. No compromises.

## Hardware & OS

I didn't want to run a server on my gaming PC — always-on hardware burns out fast. So I headed to eBay and picked up a Lenovo ThinkCentre M720s for cheap.

![Lenovo ThinkCentre M720s - eBay Listing](/media/MediaServerBuild/lenovo-ebay-listing.jpg "Lenovo ThinkCentre M720s Intel i5 8th Gen w/ Quick Sync")

ThinkCentre models are business-grade and built to last. More importantly, this one ships with **Intel Quick Sync** — Intel's hardware video transcoding engine. That means I can transcode 4K streams without a dedicated GPU, which is critical in a small form factor machine with limited expansion options.

Once it arrived, I made a few upgrades:
1. Installed a 512GB SSD as a dedicated boot drive
2. Swapped the factory HDD for an 8TB IronWolf
3. Added a 16GB Ripsaw DDR4 stick (24GB total)
4. Repasted the CPU — who knows how old the original paste was

One thing I got wrong: I assumed the M720s had two 3.5" drive bays. It doesn't — one 3.5" and one 2.5". That killed my plans for drive mirroring, so instead I ordered an 8TB external for backups via rsync.

![Seagate Expansion Desktop Drive](/media/MediaServerBuild/HDD-external-ebay.jpg "Seagate Expansion Desktop Drive 8TB USB 3.0")

For the OS, I went with [Proxmox VE](https://proxmox.com/en/products/proxmox-virtual-environment/overview) — an enterprise-grade, open source hypervisor that lets me run Linux Containers and Virtual Machines on the same host. Flashed the ISO to a USB with [Rufus](https://rufus.ie/en/), ran through the installer, and was in the web UI within 10 minutes.

![Proxmox Dashboard](/media/MediaServerBuild/proxmox-dashboard.jpg "Quick Snapshot of Proxmox VE Dashboard")

*ignore the '16GB of RAM' showing — I originally swapped the factory 8GB stick instead of adding to it. Realized my mistake and just ran both. 🤷🏼‍♂️*

## Virtualization Setup

Proxmox gives you two options for running isolated workloads: **LXC containers** and full **Virtual Machines**. Choosing the right one for each service matters.

For media storage and streaming, I went with LXCs. Since Jellyfin needs direct access to the Intel Quick Sync hardware transcoder (`/dev/dri/renderD128`), LXCs are the better fit — they share the host kernel, which makes hardware passthrough significantly simpler than fighting with VM driver layers.

For the automation stack, a VM made more sense. Docker technically runs in LXCs, but it requires privileged access or nested virtualization which introduces its own complexity. A dedicated Ubuntu Server VM keeps things clean.

Both LXCs are **unprivileged** — a deliberate security decision. Unprivileged containers map the root user inside the container to a non-root user on the host, meaning a container escape doesn't hand an attacker root on the machine. No service here needed privileged access, so there was no reason to grant it. Least privilege in practice.

| Name             | Type | Cores | RAM  | Disk  | OS            |
|------------------|------|-------|------|-------|---------------|
| media (100)      | LXC  | 2     | 2GB  | 64GB  | Ubuntu        |
| jellyfin (101)   | LXC  | 4     | 3GB  | 32GB  | Ubuntu        |
| media-apps (102) | VM   | 4     | 12GB | 64GB  | Ubuntu Server |

The automation VM gets 12GB because multiple containerized services — a VPN client, download client, and several media management tools — all run simultaneously and are actively processing downloads. 
Docker and Ubuntu Server themselves are lightweight; the headroom is for the workload.

Once everything was running, I installed OpenSSH on the PVE node and added my public key. From there I can SSH into any container or VM through the node — no need to install OpenSSH everywhere individually.

![OpenSSH](/media/MediaServerBuild/openssh-console.jpg "SSH into the node, manage the entire datacenter from one connection")

## Media Storage & Streaming

Two things need to be configured before Jellyfin can stream anything:

- Mount the **tank** (a ZFS pool currently running on the 8TB IronWolf — expandable as I add more drives) to both LXCs
- Pass through the Intel QuickSync device (`/dev/dri/renderD128`) to the Jellyfin LXC

One important distinction: the media LXC gets **read/write** access to the tank since it manages the files, but Jellyfin only gets **read-only**. Jellyfin's job is to stream — it has no business writing to the disk, so it doesn't get that permission. Least privilege again.

Here's a quick snapshot of both container configs:
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

The Jellyfin config is longer because I initially launched it as a *privileged* container and had to convert it after the fact — which required manually updating the privilege mapping translations. Lesson learned: get the security model right before you build on top of it.

With storage mounted and the iGPU passed through, installing Jellyfin is straightforward using their official script:
```bash
curl https://repo.jellyfin.org/install-debuntu.sh | sudo bash
```

After logging in and creating credentials, head to **Playback → Transcoding** to enable hardware transcoding.

![Jellyfin Settings Pt.1](/media/MediaServerBuild/jellyfin-transcoding.jpg "Navigate to Playback > Transcoding")

Select your iGPU and specify the device path. For codecs, I enabled only **HEVC, H264, and HEVC 10-bit** — these cover virtually every modern video format. Enabling everything tends to cause compatibility issues, so stick to what you actually need.

![Jellyfin Settings Pt.2](/media/MediaServerBuild/jellyfin-transcoding2.jpg "Most newer movies include HDR")

If your library includes HDR content, enable tone mapping as well — without it, HDR scenes will be completely blown out on SDR displays.

{{< callout >}}
Hardware transcoding is **crucial** on this machine. No dedicated GPU, small form factor, limited expansion options. Intel QuickSync on 8th Gen or later can handle up to 3 simultaneous 4K streams — more than enough for personal use.
{{< /callout >}}

## Download Pipeline & VPN Networking

On the media-apps VM, I installed Docker and deployed the entire stack via a single Docker Compose file. The most interesting architectural decision here is how network isolation is handled.

Rather than giving each container its own network access, all download traffic is funneled through a single **Gluetun** VPN container. Gluetun acts as a network gateway — other containers that need VPN protection use `network_mode: service:gluetun` in the compose file, which means they share Gluetun's network namespace entirely. They have no independent network interface.
```yaml
  nzbget:
    network_mode: service:gluetun  # all traffic routes through gluetun

  gluetun:
    healthcheck:
      test: ping -c 1 www.google.com || exit 1
      interval: 20s
      timeout: 10s
      retries: 5
```

The kill switch is implicit in this design — if Gluetun goes down and fails its healthcheck, any container depending on it loses network access entirely. There's no fallback to the host network. No leaks.

Containers that don't need VPN protection (like the media management services) sit on a separate internal Docker network with static IPs, isolated from the VPN namespace.

Once media is downloaded and organized, it lands in the shared `/data` volume where Jellyfin picks it up automatically. No manual intervention required.

## Networking & Public Access

![Network Diagram](/media/MediaServerBuild/net-diagram.jpg "My first network diagram attempt ever" )

### Local Network

All containers and VMs are assigned static IPs to keep the infrastructure organized and predictable. My ISP's DHCP scope starts at `192.168.1.100`, so the lower range is free for manually managed devices — no conflicts, and the pool stays open for less critical devices like phones and laptops.

| Name       | ID   | IP Address  |
|------------|------|-------------|
| pve        | Node | 192.168.1.2 |
| media      | 100  | 192.168.1.3 |
| jellyfin   | 101  | 192.168.1.4 |
| media-apps | 102  | 192.168.1.5 |
| nginx      | 104  | 192.168.1.7 |

For DNS I went with Cloudflare (`1.1.1.1`, `1.0.0.1`) — faster than most ISP resolvers and more privacy-conscious than the alternatives. Wired on Cat6a throughout.

To make library management easier, I set up a Samba share on the media LXC so I can browse and manage files directly from Windows without touching the console every time.
```bash
sudo apt install samba
sudo nano /etc/samba/smb.conf
```
```ini
[media]
   path = /data
   browseable = yes
   read only = no
   guest ok = no
```
```bash
sudo smbpasswd -a nick
sudo systemctl restart smbd
```

![Windows Network Share](/media/MediaServerBuild/net-share.jpg "SMB share mounted on Windows for easy library management")

---

### Public Access — Nginx Reverse Proxy

With the local network sorted, the next challenge is secure public access. Everything is plaintext by default, and not all services should be treated equally — Jellyfin is meant to be publicly accessible, but admin panels like Proxmox and Nginx itself should never be exposed directly to the internet.

The approach:
- **Public-facing services** (Jellyfin) → Nginx reverse proxy with SSL via Let's Encrypt, port 443
- **Admin panels** (Proxmox, Nginx) → Cloudflare Zero Trust tunnel, no open ports required

First, I purchased a domain through Cloudflare for under $12 and configured a proxied A record pointing to my home IP. Cloudflare's proxy means third parties never see my real IP — they only see Cloudflare's.

![Cloudflare Domain Purchase](/media/nginx-setup/domain-purchase.jpg 'nix-forge.com')

![DNS Config](/media/nginx-setup/config-dns-record.jpg 'Proxied A record — real IP stays hidden')

I spun up a dedicated Ubuntu Server VM for Nginx and deployed Nginx Proxy Manager via Docker Compose alongside a Cloudflare DDNS container that automatically updates the DNS record if my home IP ever changes.
```yaml {filename="docker-compose.yaml"}
services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "81:81"
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt

  cloudflare-ddns:
    image: favonia/cloudflare-ddns:latest
    container_name: cloudflare-ddns
    restart: unless-stopped
    environment:
      - CF_API_TOKEN=${CF_API_TOKEN}
      - DOMAINS=your-domain
      - PROXIED=true
```

{{< callout type="warning" >}}
Always store API keys and sensitive values in a `.env` file. Never hardcode them directly in `docker-compose.yaml`.
{{< /callout >}}

With NPM running on port 81, the first step is generating a wildcard SSL certificate via Let's Encrypt DNS-01 challenge. This covers all subdomains under your domain without requiring port 80.

Navigate to **Certificates → Add Certificate**, add `*.yourdomain.com` and `yourdomain.com`, select Cloudflare as the DNS provider, and enter your API token.

![Certificate Creation](/media/nginx-setup/letsencrypt.jpg "Wildcard cert covers all subdomains automatically")

Then add a proxy host for Jellyfin. Forward port 443 on your router to the nginx VM IP first, then configure the host in NPM:

![Proxy Host Settings 1](/media/nginx-setup/proxy-host.jpg)
![Proxy Host Settings 2](/media/nginx-setup/proxy-settings1.jpg "Subdomain, internal IP, and port")
![Proxy Host Settings 3](/media/nginx-setup/proxy-settings2.jpg "Force SSL, HTTP/2, HSTS, and Websockets all enabled")

Enable **Websockets Support** — Jellyfin requires it for playback. Enable **Block Common Exploits**, **Force SSL**, and **HSTS** while you're there.

![Jellyfin Reverse Proxy](/media/nginx-setup/jellyfin-nginx.jpg "Jellyfin accessible via HTTPS from anywhere")

---

### Admin Security — Cloudflare Zero Trust Tunnel

Exposing admin panels directly on port 443 is a risk — even with a login page, the surface is still publicly reachable. Cloudflare Zero Trust eliminates that entirely.

The key difference:
- **Nginx proxy** — port 443 open, traffic flows through, app login is the only protection
- **Zero Trust tunnel** — no ports open, Cloudflare enforces identity verification *before* traffic ever reaches your server

The tunnel works by running a lightweight `cloudflared` daemon on the nginx VM that maintains a persistent outbound connection to Cloudflare. No inbound ports required.
```bash
# Install cloudflared
curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared.deb

# Authenticate and install as a service
sudo cloudflared service install <tunnel-token>
sudo systemctl enable --now cloudflared
```

With the tunnel running, each admin service gets its own public hostname route configured in the Cloudflare Zero Trust dashboard — pointing directly to the internal IP and port. No NPM proxy host needed.

| Service  | Tunnel Route            | Internal Target       |
|----------|-------------------------|-----------------------|
| Proxmox  | pve.nix-forge.com       | https://192.168.1.2:8006 |
| NPM      | npm.nix-forge.com       | http://192.168.1.7:81 |

For Proxmox specifically, **No TLS Verify** must be enabled in the tunnel origin settings — Proxmox uses a self-signed certificate that the tunnel would otherwise reject.

Each route is then protected by a Cloudflare Access policy requiring email verification before the tunnel will forward any traffic. The result: admin panels are completely invisible to port scanners, and unreachable without passing Cloudflare's auth wall first.

{{< callout type="info" >}}
Only port **443** needs to be forwarded on your router — and only for Nginx/Jellyfin. The Zero Trust tunnel is entirely outbound and requires no port forwarding at all.
{{< /callout >}}

## Reflecting on Challenges & Lessons

### Linux Permissions & the iGPU Rabbit Hole

Coming into this project, my Linux experience was limited to basic networking commands. That changed fast.

The first wall I hit was file permissions. Jellyfin was failing to access `/dev/dri/renderD128` and the error output was nearly useless. I didn't know how to read permission strings like `drwx---r-x`, let alone diagnose why a device node was inaccessible to a specific user inside an unprivileged container.

`journalctl -xe` became my best friend. Correlating the Jellyfin service logs with kernel-level device access errors pointed me toward a group membership issue — the container's render group GID wasn't mapped correctly to the host. Once I understood how Linux permission bits and group ownership actually interact, the fix was straightforward. Getting there wasn't.

### Hardware Passthrough: VMs vs LXCs

My original architecture had Jellyfin running as a Docker container inside the same VM as everything else. Passing `/dev/dri/renderD128` through to a Docker container inside a VM meant traversing two abstraction layers — the hypervisor and the container runtime — and getting the device permissions, cgroup entries, and render group mappings right across both proved to be a moving target.

After days of ambiguous errors I went back to the Jellyfin documentation and found the answer I'd overlooked: Jellyfin explicitly recommends LXCs for hardware transcoding. LXCs share the host kernel directly, which makes device passthrough significantly simpler — you're configuring one layer, not two.

The architectural lesson was more valuable than the fix itself: **choose your virtualization primitive based on what the workload actually needs**. Docker-in-VM is great for networked services. Hardware passthrough belongs in a container.

You can verify QuickSync is actively transcoding by monitoring CPU usage during playback — it should stay near idle. For more detail, `intel_gpu_top` shows GPU engine utilization in real time.
```bash
sudo apt install intel-gpu-tools
sudo intel_gpu_top
```





This project started as a simple media server and turned into a full infrastructure build —
hypervisor setup, container architecture, VPN networking, reverse proxying, and Zero Trust
security, all running on a $200 eBay machine. The homelab is still expanding. Next up:
monitoring stack, automated backups, and whatever rabbit hole comes after that.