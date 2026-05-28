---
title: "Surfer"
date: 2026-05-22
difficulty: Medium
os: Linux
topics: ["SSRF", "Web Application"]
tags: ["tryhackme"]
---
{{< badge content="Easy" color="green" >}}
{{< badge content="Linux" color="blue" >}}
{{< badge content="SSRF" color="green" >}}
{{< badge content="Web Application" color="green" >}}

# Overview
A simple 
## Reconnaissance

Per instructions, I navigated to http://<ip> - I was met with this page. Not much to go off.

![Login page](/media/labs/surfer/login.png)

Not much to go off here. 

## Enumeration

Since all I had to go on was this login page, I decided to cut straight to the chase: Gobuster. 

I used `gobuster dir -u http://<ip-addr> -w /usr/share/wordlists/dirb/common.txt -x txt,php,js,html,old,bak`

Result came back for:

![Gobuster Results](/media/labs/surfer/gobuster-results.jpg)

Right off the bat, I was interested in:
- /robots.txt
- /server-info.php
- /changelog.txt
- /ReadMe.txt
- /index.php
- /backup

## Vulnerability Analysis

Upon checking /robots.txt - I found this:

![Robots.txt Results](/media/labs/surfer/robots.jpg)

Obviously quite interesting. 

When nav'ing to /backup/chat.txt, there seemed to be some unfinished conversation:

![Backup Chat](/media/labs/surfer/backup-chats.jpg)

So we found out password: admin / admin (I guess I could have just tried that to begin with)

Immediately upon login, I noticed the 

![Backup Chat](/media/labs/surfer/backup-chats.jpg)

Scrolling down (same page), I found this "Export PDF" button, which would clearly be our target for SSRF.

![Logged in, found export](/media/labs/surfer/logged-in.jpg)

Looks like we found our target 

Inspect > Network > Requests shows me the export action is requesting from `http://127.0.0.1/server-info.php`

![Network Requests](/media/labs/surfer/network-requests.jpg)

This is a perfect setup for SSRF. 

## Exploitation

Opened my 


## Post-Exploitation
Privilege escalation, find flags, pivot if needed.