---
layout: post
title: HTB Nocturnal Writeup
subtitle: Detailed overview of exposed credentials, command injection, and exploiting known application vulnerabilities to gain root access.
tags: [htb, linux, writeup]
thumbnail-img: /assets/img/thumb1.png
author: ad3n
---

Nocturnal is a Linux-based machine categorized as easy difficulty. This post provides a step-by-step walkthrough to gain root access on the machine.

## Scanning & Enumeration

### NMAP Scan

{% highlight bash linenos %}
└─$ nmap -A -sC -sV -Pn 10.10.11.64

Starting Nmap 7.95 ( https://nmap.org ) at 2025-05-08 06:30 EDT
Nmap scan report for nocturnal.htb (10.10.11.64)
Host is up (0.017s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 20:26:88:70:08:51:ee:de:3a:a6:20:41:87:96:25:17 (RSA)
|   256 4f:80:05:33:a6:d4:22:64:e9:ed:14:e3:12:bc:96:f1 (ECDSA)
|_  256 d9:88:1f:68:43:8e:d4:2a:52:fc:f0:66:d4:b9:ee:6b (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-title: Welcome to Nocturnal
Device type: general purpose|router
Running: Linux 4.X|5.X, MikroTik RouterOS 7.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
OS details: Linux 4.15 - 5.19, MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
{% endhighlight %}

In summary, there is only 2 open ports which consist of `ssh` and `http`. Since the ssh port is on the latest version, we will shift our target to http and fuzz the directory to discover any interesting endpoint.

{: .box-note}
**Note:** Add `10.10.11.64` in `/etc/hosts` file and named it to `nocturnal.htb`.

![Webnocturnal](/assets/img/webnocturnal.png){: .mx-auto.d-block :}

## Directory Enumeration

{% highlight bash linenos %}
└─$ gobuster dir -u http://nocturnal.htb/ -w /usr/share/wordlists/dirb/common.txt                                                                   
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://nocturnal.htb/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/admin.php            (Status: 302) [Size: 0] [--> login.php]
/backups              (Status: 301) [Size: 178] [--> http://nocturnal.htb/backups/]
/index.php            (Status: 200) [Size: 1524]
/uploads              (Status: 403) [Size: 162]
Progress: 4614 / 4615 (99.98%)
===============================================================
Finished
===============================================================
{% endhighlight %}


