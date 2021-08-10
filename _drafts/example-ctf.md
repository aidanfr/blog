---
layout: post
title: TryHackMe | Fowsniff CTF
tags: [ctf, boot2root, beginner]
---

I recently did a classic boot2root box off TryHackMe made by [@berzerk0](https://twitter.com/berzerk0). After completing it, I thought it would be a great opportunity to talk about some 101 level stuff instead of diving into explaining a four hour long HackTheBox Hard. Hopefully this helps answer some questions people might have been too afraid to ask.

-----

### Initial Recon and Some Networking 101

[Breakdown of TCP and the three way handshake]

Like every CTF, the first thing I did after getting my target machine's IP address was run an `nmap` scan. Nmap is a network scanner that allows us to identify hosts, find open ports on hosts, and even find some important characteristics on hosts like operating system or network service details. This is by no means an exhaustive list of nmap's capabilities but for this walkthrough we'll focus on its port discovery and service discovery capabilities. With our target IP, I ran my run this initial (and only) scan:

```bash
> nmap [target-ip] -sV -O
```

Let's break this command down and talk about what I'm actually doing here. Because we haven't specified which type of scan we want to run, by default we are executing a TCP SYN scan against our target against the first 1000 ports. A TCP SYN scan works by sending a TCP packet with the SYN flag set to each port where three outcomes are possible:

* If the port is open, the target host will send a response with the SYN and ACK flags back, which our host will then respond to with a TCP packet with a RST flag. The reason it responds with RST instead of completing the three-way handshake is that because the target host has sent a SYN / ACK response, we know the port is open and we do not want to have to complete another three-way handshake to close the connection. 

* If the port is closed, the target host will respond back with a TCP packet with a RST flag. 

* If the port is filtered, the target host won't respond back with anything, which can mean a lot of things, but usually means the port is behind a firewall or the host is down. 


The `-sV` flag in our command attempts to determine the version of the service running on each open port and the `-O` flag attempts to determine identify what operating system is running on the host. This information can be very valuable when it comes to the later stages of enumeration, exploitation, and even privilege escalation. Building up as much information during the active reconaissance stage helps us down the road when we start the enumeration process and even privilege escalation upon initial access once inside the target. 

Nmap responded back with this scan report: 

```bash
Nmap scan report for [target-ip]
Host is up (0.22s latency).
Not shown: 996 closed ports
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http    Apache httpd 2.4.18 ((Ubuntu))
110/tcp open  pop3    Dovecot pop3d
143/tcp open  imap    Dovecot imapd
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```


### Port 80 Is Always a Gold Mine (Well Not Really But It's Always Good To Check)


My initial reaction upon seeing the Nmap scan results was to go to my browser and type in the target IP address into the searchbar. Why? Because port 80, a port reserved for http, the protocol that makes most of the seeable Internet just werk, was open with potentially even an Apache http server running. Lo and behold, there was a company website live with a notice, saying that they recently had a data breach and that their company twitter was defaced by some *not-so-nice* hackers.

![Company Website](/assets/fowsniff-ctf-assets/website.png)

After thanking my friends for doing the hard work, I went to their twitter page and found a link to a pastebin with some interesting text files.

![Defaced Company Twitter](/assets/fowsniff-ctf-assets/company-twitter.png)

### Intro to Hashing / Cracking Bad Passwords with hashcat

[What is a Hash function / what used for]
[Why md5 is broken]
[what is hashcat]
[how to use hashcat]

### Enumerating POP3 and Gaining Initial Access

[explain POP3]
[telnet]
[basic pop3 commands]
[getting company email]
[ssh'ing into target]


### From `user` to `root` (Privilege Escalation Basics)

[explain users and groups]
[explain perms]
[explain shell scripting]
[explain sudo vs root]
[explain reverse shell]

### Closing Thoughts

[overview]
[hint at second priv esc route]