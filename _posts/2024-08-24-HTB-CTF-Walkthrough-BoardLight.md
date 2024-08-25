---
layout: post
title: HTB CTF Walkthrough. BoardLight
date: 2024-08-24
categories: [CTF, HackTheBox]
tags: [ctf, htb, walkthroughs]
description: "This post provides a comprehensive guide to solving the BoardLight challenge on HackTheBox, including all steps and strategies."
keywords: 
  - CTF
  - HackTheBox
  - BoardLight challenge
  - Cybersecurity walkthrough
  - BoardLight
---

![logo](/assets/images/boardlight/1.png)

## Introduction

In this walkthrough, I will guide you through the steps to compromise the "BoardLight" machine on Hack The Box. This is Easy-difficulty machine requires a mix of enumeration, exploiting web vulnerabilities, and privilege escalation techniques. By the end, you’ll gain root access and capture the flag.

## User Flag

Let's start with an `nmap` scan to identify open ports and services running on the target machine.

```bash
nmap -sC -sV <target_ip>
```
The results show: Port 22 (SSH) and Port 80 (HTTP).

![](/assets/images/boardlight/2.png)

The site opened without any problems using the IP address.
At the footer we find a email with the domain `board.htb`. Let's added it to `/etc/hosts`.

Now let's check subdomains by `wfuzz` tool. After checking we found `crm` record. Let's add it too to `/etc/hosts`.
> ** I added special filter `--hl 517` to revome unnessesary subdomains that redirect to main page (517 number of the Lines into page)

> ** I am using special word lists [seclists](https://github.com/danielmiessler/SecLists)

```bash
wfuzz -c -u 'http://board.htb' -H "Host:FUZZ.board.htb" -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-20000.txt --hl 517
```

![](/assets/images/boardlight/3.png)

cat /etc/hosts
```bash
10.10.11.11     board.htb, crm.board.htb
```

Browsing to `http://crm.board.htb` we see a logon page. 

Don't forget to try simple entry options :) There are `admin/admin` creds.

![](/assets/images/boardlight/4.png)

Also note the name of the application and its version at one time. `Dolibarr 17.0.0`

After a short search on the Internet, we find a [CVE](https://github.com/nikn0laty/Exploit-for-Dolibarr-17.0.0-CVE-2023-30253) with a prepared exploit.

Read `exploit description`, prepare the `NetCat` for `revers-shell` and run the exploit (as we have all the needed information).

```bash
netcat -nvlp 5555
```

![](/assets/images/boardlight/5.png)


Reverse shell received as `www-data`.

We see `www-data@boardlight:~/html/crm.board.htb/htdocs$` there a lot of directories. After brows I found `conf` directory with 3 conf files. Checked all files and we get `ssh password` in `conf.php` file. 

![](/assets/images/boardlight/6.png)

Let's check the username of User `larissa` (find it into `/etc/passwd`) and just resived password. And as we see `larissa` used the same password for Application DB and for ssh. Getting the `user flag`

![](/assets/images/boardlight/7.png)

## Root Flag

One of my favorite scripts for a Privilege Escalation for Linux is ["LinPEAS"](https://github.com/peass-ng/PEASS-ng/tree/master/linPEAS).
Upload it to the server and run.

This cool little peas will tell us that there is a possible privilege escalation via SUID on the server. 

![](/assets/images/boardlight/8.png)

We could also see it with the next command.

```bash
find / -perm -4000 -type f 2>/dev/null
```
![](/assets/images/boardlight/9.png)

Just a litle googling, around found DLL and BIN files, I found [CVE](https://www.exploit-db.com/exploits/51180) into `Enlightenment`.

Also here you could find [exploit](https://github.com/MaherAzzouzi/CVE-2022-37706-LPE-exploit/blob/main/README.md)

You could read `exploit description`, but also just download and run it.
Getting the `root flag`

![](/assets/images/boardlight/10.png)