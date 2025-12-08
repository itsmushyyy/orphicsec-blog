---
title: "HTB - Editor"
date: 2024-12-08
---

| **OS:** Linux

| **Difficulty:** Easy [20]

| **Season:** 9

---

**Editor** is a Linux box hosting a code editor website, with documentation on an XWiki instance. I’ll exploit a vulnerability in XWiki’s Solr search that allows unauthenticated Groovy script injection to get remote code execution and a shell. From there, I’ll find database credentials in the XWiki Hibernate config and pivot to a user who reuses the password. Enumerating localhost services, I’ll find NetData running an older version that installs a vulnerable ndsudo SetUID binary that is vulnerable to PATH injection, which I’ll abuse to get root.

# Recon
## Initial Scanning
`nmap` finds three open TCP ports, SSH (22) and two HTTP (80, 8080):
```bash
oxdf@hacky$ nmap -p- -vvv --min-rate 10000 10.10.11.80
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-08-09 18:01 UTC
...[snip]...
Nmap scan report for 10.10.11.80
Host is up, received echo-reply ttl 63 (0.12s latency).
Scanned at 2025-08-09 18:01:57 UTC for 16s
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE    REASON
22/tcp   open  ssh        syn-ack ttl 63
80/tcp   open  http       syn-ack ttl 63
8080/tcp open  http-proxy syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 16.11 seconds
           Raw packets sent: 153968 (6.775MB) | Rcvd: 105117 (4.205MB)
oxdf@hacky$ nmap -p 22,80,8080 -sCV 10.10.11.80
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-08-09 18:06 UTC
Nmap scan report for 10.10.11.80
Host is up (0.089s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp   open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://editor.htb/
8080/tcp open  http    Jetty 10.0.20
| http-methods:
|_  Potentially risky methods: PROPFIND LOCK UNLOCK
|_http-server-header: Jetty(10.0.20)
| http-webdav-scan:
|   Allowed Methods: OPTIONS, GET, HEAD, PROPFIND, LOCK, UNLOCK
|   Server Type: Jetty(10.0.20)
|_  WebDAV type: Unknown
| http-title: XWiki - Main - Intro
|_Requested resource was http://10.10.11.80:8080/xwiki/bin/view/Main/
| http-cookie-flags:
|   /:
|     JSESSIONID:
|_      httponly flag not set
| http-robots.txt: 50 disallowed entries (15 shown)
| /xwiki/bin/viewattachrev/ /xwiki/bin/viewrev/
| /xwiki/bin/pdf/ /xwiki/bin/edit/ /xwiki/bin/create/
| /xwiki/bin/inline/ /xwiki/bin/preview/ /xwiki/bin/save/
| /xwiki/bin/saveandcontinue/ /xwiki/bin/rollback/ /xwiki/bin/deleteversions/
| /xwiki/bin/cancel/ /xwiki/bin/delete/ /xwiki/bin/deletespace/
|_/xwiki/bin/undelete/
|_http-open-proxy: Proxy might be redirecting requests
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.63 seconds
```

Based on the [OpenSSH and nginx](https://0xdf.gitlab.io/cheatsheets/os#ubuntu) versions, the host is likely running Ubuntu 22.04 jammy [LTS].

All of the ports show a TTL of 63, which matches the [expected TTL](https://0xdf.gitlab.io/cheatsheets/os#os-identification) for Linux one hop away.

The website on port 80 is redirecting to `editor.htb`.

## Subdomain Brute Force

Given the use of domain name based routing and the domain `editor.htb`, I’ll check for subdomains that respond differently than the default case using `ffuf`:
```bash
oxdf@hacky$ ffuf -u http://10.10.11.80 -H "Host: FUZZ.editor.htb" -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -ac

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.10.11.80
 :: Wordlist         : FUZZ: /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt
 :: Header           : Host: FUZZ.editor.htb
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

wiki                    [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 93ms]
:: Progress: [19966/19966] :: Job [1/1] :: 447 req/sec :: Duration: [0:00:45] :: Errors: 0 ::
```





















