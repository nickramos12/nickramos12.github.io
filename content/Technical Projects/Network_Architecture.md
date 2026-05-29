---
title: Zero Trust Architecture v1.0
weight: 3
params:
  images:
    - /media/projects/zero_networkv1/wFeoN.jpg
---
#### TLDR

Designed and implemented a segmented Zero Trust-inspired network architecture for a self-hosted media server environment.
Configured strict micro-segmentation across 6 VLANs, deployed reverse proxy with WAF, Cloudflare Tunnel for secure remote access, and compensating controls (WireGuard VPN, MAC port security, BPDU Guard).
Reduced attack surface by enforcing least-privilege firewall rules and isolating high-risk devices.
Demonstrates practical application of network security principles, firewall management, and secure remote access.


## Introduction

Now that my media [server is live](https://nickramos12.github.io/technical-projects/private_server_build/) and functional, I can only think of how to improve it. 
My first thought is, "What if I want to access this while traveling?" or "What if I want to share this with my dad, but he lives across the country?"

I realized that if I wanted to expose applications to internet, I needed a more advanced network setup than what Google Fiber provides their customers. 

So for this project, I'm taking my first step towards implementing a Zero Trust Network Architecture.

#### Project Goals
- Segment devices based on risk profile
- Configure secure remote access for applications
- Implement strict firewall rules
- Improve layer 2 security
- Harden the overall system

For my next project, I'll tackle 802.1X authentication, SIEM implementation, and configuring the network for IPv6 for efficient routing.

## Architecture Overview

While designing the network, my main goals were **scalability** and **strong security through risk-based segmentation**.

| Network       | Purpose                                                                 |
|---------------|-------------------------------------------------------------------------|
| Management    | Network infrastructure, controllers, and management interfaces          |
| Applications  | Self-hosted services (media, automation, productivity, etc.)            |
| DMZ           | Public-facing services accessed through reverse proxy                   |
| Workstations  | User devices (laptops, tablets, etc.) that regularly leave the network  |
| IoT           | Low-trust devices (smart home, printers, TVs, etc.)                     |
| Guest         | Untrusted devices (phones, tablets, visitor laptops)                    |

**Risk-Based VLAN Segmentation Details**

- **Guest VLAN:** Designed for untrusted visitors and temporary devices. Since people remain one of the weakest links in security (phishing, public Wi-Fi usage, Bluetooth/AirDrop exposure, etc.), this segment is heavily isolated with strict outbound firewall rules. 
- **IoT VLAN:** Dedicated to low-trust smart home devices, printers, and TVs. IoT devices are notoriously insecure and frequent targets of supply chain attacks. I observed multiple devices attempting unauthorized outbound connections (often to random cloud endpoints), which are now blocked by default at the firewall. 
- **DMZ (Screened Subnet):** Houses all public-facing services and the reverse proxy. This creates a controlled buffer zone between the internet and the internal network, ensuring that even if a publicly accessible service is compromised, attackers cannot easily pivot inward. 
- **Workstations VLAN:** For user devices such as laptops, desktops, and tablets that regularly leave the network and return. These are treated as “dirty” due to potential exposure to external threats. 
- **Applications VLAN:** Contains core self-hosted services (media server, automation tools, etc.) — protected from higher-risk segments. 
- **Management VLAN:** Reserved exclusively for infrastructure devices (router, switch, WAP, servers). This critical segment is the most tightly controlled.

This segmentation strategy follows Zero Trust principles of “never trust, always verify” and least privilege. By assuming devices — especially wireless ones — may already be compromised, I intentionally keep high-risk assets away from sensitive internal resources. While this initial design has room for refinement, it significantly limits lateral movement.

## Screened Subnet (DMZ)

A major driver for this project was enabling secure remote access to my self-hosted media library. Whether traveling or sharing content with family across the country, I needed a way to expose services externally without compromising the internal network.

This requirement pushed me fully into Zero Trust thinking: assume breach, never trust, and verify explicitly. With a path now open to the internet, I could no longer rely on HTTP logins or broad exposure.To secure the DMZ, I implemented the following layered controls:

**Cloudflare Tunnel + Web Application Firewall (WAF)**  
- Established an outbound-only encrypted tunnel from within the Docker network directly to Cloudflare.
- Enabled WAF rules to block risky ASNs, common exploit paths, automated scanners, and malicious traffic patterns.  
- Result: ~99% of inbound noise is blocked at the edge. I review logs weekly to refine rules.

**Zoraxy Reverse Proxy**  
- Deployed Zoraxy as the central reverse proxy in the DMZ.  
- Automatically provisioned TLS certificates for each application hostname.  
- Configured internal DNS overrides so all traffic to these hostnames is forced through Zoraxy over HTTPS.

This combination provides secure, authenticated remote access while maintaining a strong isolation boundary. Full end-to-end encryption and additional hardening (e.g., mTLS) are planned for the next phase.

## Core Components

My ISP (Google Fiber) has great speeds, but default gateway they provide is extremely limited in terms of configuration access - so I decided to grab fresh hardware.

![Internal Snapshot](/media/projects/zero_networkv1/physical_stack.jpeg)

**Router:**
Used Lenovo M720s running OPNsense. Intel i5-8400 @ 2.80GHz (6 Cores), 1TB HDD, 256GB SSD Boot Drive, 8GB Memory. 
Yes the specs are overkill - I'll likely move this to a mini PC in the future, but I need to see how consumption changes when enabling additional services.

**Switch:**
I went with the Lite 8 PoE Managed Switch straight off the UniFi website. It comes with 8 ports, 4 of those ports supporting up to POE+ (802.3at).
Perfect for what I need.

**Wireless Access Point:**
Picked up a UniFi 6 Lite Wi-Fi 6 Access Point. Came with Dual-Band MIMO - it supports 2.5Ghz & 5GHz. In my next project, we'll make the jump to 6GHz WAP. 

**Cat6a Custom Cabling:**
I figured it wouldn't be a bad idea to learn how to correctly terminate cables. So I ordered a [cheap network repair kit](https://www.amazon.com/dp/B0756SN86D) off of Amazon, and paired it with 500ft of Cat6a (STP).


## Layer 2 Security Controls

UniFi devices can be controlled from a self-hosted application called UniFi Controller. 

![Internal Snapshot](/media/projects/zero_networkv1/unifi_dash.jpg)

**Disabled Unused Ports:**
Unused ports are open season for any attacker who gains physical access to my system. I disabled the ports and turned off POE to conserve power. 

**Restricted Access Ports by MAC Address:**
For fixed ports that should never have more than a single device, I restricted access to that specific MAC address. 

**Enabled BDPU Guard for all Access Ports:**
This is just a common sense configuration. Although I've already restricted ports by MAC address, I want to eliminate the risk of access ports being used as unauthorized trunk ports. 

**Strong Wi-Fi Passphrases:**
All wireless networks use 35+ character passphrases generated with high complexity and stored in Bitwarden. IoT network uses WPA2, Guest uses WPA2/WPA3.

**Enabled WPA2 for IoT Network, and WPA2/WPA3 for Guest Network:**
Most IoT devices will have issues with WPA3, so I set the IoT network to WPA2 to prevent issues (for now), and the Guest Network is configured for WPA2/WPA3 which would mean if the device is WPA3 capable - that's what it'll use. 

**Moved Devices to Management VLAN:**
Both the switch and WAP were moved to a more secure subnet, putting the sub-interface (and firewall) in between attackers and potential SSH access to infrastructure devices. 

I only have a single switch, so spanning tree protocol is not needed. 

## Layer 3 Security Controls

OPNsense is the nexus of my current Layer 3 security. 

![Internal Snapshot](/media/projects/zero_networkv1/opn_dash.jpg)

**Subnet Sizing:**
After implementing my VLANs (primarily Layer 2), I decided to limit subnet sizes to the minimum required for functionality. Less available IP addresses the better.
I'll edit sizes if need be, but there's no benefit in having 254 slots when I only need 6 for the foreseeable future. 

**Strict Firewall Rules:** 
Since we segmented the network, our firewall has a much more active role in our network security. 
I implemented strict firewall rules PER device, very few broad rules. Nearly all of them require a specific destination address AND destination port for you to traverse. 
This would likely be impractical in an enterprise environment, but since this is just my local network - I can adjust on the fly quite easily. 

**Centralized NTP & DNS:**
I forced all of my VLANs to forward network time requests to OPNsense trusted internal NTP server. I also forced all DNS requests to forward upstream. 
Reduces attack surface on each device (no direct DNS cache poisoning, amplification, or C2 channels from network hosts).

**WireGuard Server:** All IoT devices are in a dedicated high risk VLAN, but devices like such only use WPA2 - so I decided to at least pair it with a VPN (common compensating control).
If an attacker does break WPA2 encryption, then they'll have to break the VPN before they can see anything of use in traffic - plus they're trapped inside the IoT network. 



## Additional Hardening

Beyond the core Layer 2 and Layer 3 controls, I applied several additional hardening measures across the environment:

- Disabled SSH root login on all systems and enforced key-based authentication with strong passphrases  
- Configured high-entropy, unique passwords for every service and device (managed via Bitwarden)  
- Enabled automatic security updates and unattended-upgrades on Ubuntu hosts  
- Set strict file permissions and disabled unnecessary services/daemons across servers  
- Physically secured the rack/stack in a locked room with limited access

These steps further reduced the attack surface and aligned the lab with defense-in-depth principles.



## Challenges & Lessons Learned

This project was a crash course in real-world network security trade-offs. Several issues tested my troubleshooting skills and reinforced important lessons:

- **Firewall Rule Lockouts:** Strict per-device, destination-IP-and-port rules frequently locked me out of my own systems. This taught me the critical importance of always maintaining an out-of-band management path (e.g., physical console access or a dedicated “break-glass” rule) and testing changes incrementally.
- **NFS Race Conditions Across VLANs:** Moving services between segmented networks introduced NFS mount failures due to timing and routing issues. I learned to carefully plan inter-VLAN dependencies and use firewall rules more surgically instead of relying on broad “allow all” shortcuts.
- **Native VLAN Confusion:** Misunderstanding the default/native VLAN behavior on the UniFi switch caused initial connectivity problems when moving management interfaces. This highlighted how Layer 2 fundamentals can quietly break higher-level configurations if not fully documented.
- **Performance vs. Security Balancing:** Overly aggressive subnet sizing and firewall rules occasionally impacted service responsiveness (especially media streaming). I now better understand the need to measure before and after changes and document acceptable performance baselines.

Overall, the biggest takeaway was that Zero Trust is easy in theory but operationally demanding in practice. Assuming breach forces you to think through every dependency, and small configuration mistakes can have outsized effects. These experiences dramatically improved my systematic troubleshooting and documentation habits.



## Conclusion

This project marks a significant milestone in my cybersecurity journey. Just a few months ago, I was still getting comfortable with the Ubuntu CLI. Today, I am confidently designing and hardening a segmented Zero Trust-inspired network architecture, routinely evaluating complex trade-offs in subnet sizing, Layer 2 security controls, VPN compensating controls for WPA2 devices, and physical attack vectors.

What started as basic command-line exploration has become one of the most engaging and valuable hands-on experiences I’ve ever had. This project taught me far more than studying for the Network+ exam, giving me real comfort and confidence in building secure network topologies from the ground up.

Building a resilient Zero Trust environment is a continuous process. My next priorities include implementing 802.1X/RADIUS authentication, deploying a SIEM solution, and achieving full-stack HTTPS encryption across the network.

I’m actively learning and iterating — feedback and connections from the community are always welcome.

