---
layout: post
title: HTB CTF Walkthrough. Editorial
date: 2024-08-19
categories: [CTF, HackTheBox]
tags: [ctf, htb, walkthroughs]
description: "This post provides a comprehensive guide to solving the Editorial challenge on HackTheBox, including all steps and strategies."
keywords: 
  - CTF
  - HackTheBox
  - Editorial challenge
  - Cybersecurity walkthrough
  - Editorial
---

![logo](/assets/images/editorial/editorial-logo.png)

## Introduction

In this walkthrough, I will guide you through the steps to compromise the "Editorial" machine on Hack The Box. This is Easy-difficulty machine requires a mix of enumeration, exploiting web vulnerabilities, and privilege escalation techniques. By the end, you’ll gain root access and capture the flag.

## User Flag

Let's start with an `nmap` scan to identify open ports and services running on the target machine.

```bash
nmap -sC -sV -A <target_ip>
```
The results show: Port 22 (SSH) and Port 80 (HTTP).

![nmap](/assets/images/editorial/editorial-nmap.png)

When we enter the IP in the browser we can see that it is redirected to `editorial.htb` and block. So let’s add it to /etc/hosts to fetch it.

```bash
10.10.11.23     editorial.htb
```

Browsing to `http://editorial.htb` we see a basic website. We also found a upload form `http://editorial.htb/upload`.

> **Whenever it’s a web page always run through Burp suite!

Just when entering data through each of the input fields, I observed that a few seconds after entering “Book information” field a request is being sent to the server.

This is a classic sign of [SSRF](https://portswigger.net/web-security/ssrf). So we found how to put our first foot into the system, atleas the way towards it.

Now let’s enter the local IP (127.0.0.1) in the input and see what happens.

![burp-enum](/assets/images/editorial/editorial-burp-enum.png)

When attempting to download by appending the response endpoint with editorial.htb, the file retrieved contains no useful data or metadata. This suggests we might need to target a specific port. It's just a thought, but let's see where it leads. Time to bring out the heavy artillery — `Intruder` tool — to brute-force the ports.

Configure the payload type to `Numbers`, ranging from 1 to 65,535 with a step of 1. Then, initiate the ATTACK!

![burp-intruder](/assets/images/editorial/editorial-intruder.png)

After the attack completes, apply filters to isolate the results that contain the typical jpeg endpoint in the response. This should reveal port `5000` as the anomaly with a different endpoint in the response

After trying to look into downloaded file from `5000` port we could fint internal endpoints. 

![burp-enum](/assets/images/editorial/editorial-burp-enum2.png)

Now I'm preparing myself a list of endpoints where I will look for more.

![burp-enum](/assets/images/editorial/editorial-endpoints.png)

After searching through all the endpoints, into respons of `/metadata` I found the login and password for `ssh` user `dev`.

![burp-enum](/assets/images/editorial/editorial-burp-enum3.png)

Into field "Choose file" put empty file with .php

![burp-enum](/assets/images/editorial/editorial-burp-enum4.png)

Now just login via ssh and resive `user flag`

![burp-enum](/assets/images/editorial/userflag.png)

## Root Flag

In the home directory of the `dev user`, we locate the `/app` folder, and within it, the hidden `.git` directory. By running the `git log` command, we find several recent commits. After examining them, by running the `git show <commit-uuid>` we discover deleted information that includes the login and password for the `prod` user.

![burp-enum](/assets/images/editorial/git.png)

![burp-enum](/assets/images/editorial/git2.png)

Now, let's SSH into the "prod" user:

```bash
ssh prod@10.10.11.20
```

Let's try the command for find some priv escalation `sudo -l`. And we see that with sudo privileges we can run `python3` command with python code. Let’s understand how we can manipulate with it.

![burp-enum](/assets/images/editorial/sudo-l.png)

![burp-enum](/assets/images/editorial/script.png)

The `multi_options parameter` includes a Git configuration setting ("-c protocol.ext.allow=always") that allows the use of custom protocols.

If the script or any subsequent process automatically executes scripts from the cloned directory (e.g., running `ext.sh` from the `new_changes` directory without proper validation), the malicious `ext.sh` script could run with elevated privileges.

So we could craft `ext.sh` to exploit the system.

![burp-enum](/assets/images/editorial/rootflag.png)

Here we use `ext::sh -c cat% /root/root.txt% >% /tmp/lol` for injecting a shell command (cat /root/root.txt > /tmp/lol) by leveraging the ext::sh protocol.

> **I use `%` before space for a sneaky trick for escaping spaces.

The command was intended to read the contents of `/root/root.txt` and redirect it to a file named `/tmp/lol`, effectively exfiltrating the content of the root-owned file.