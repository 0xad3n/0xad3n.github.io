---
layout: post
title: HTB Nocturnal Writeup
subtitle: Detailed overview of IDOR, command injection, and exploiting known application vulnerability to gain root access.
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

In summary, there is only 2 open ports which consist of `ssh` and `http`. Since the ssh port does not have any known vulnerabilites, we will shift our target to http and fuzz the directory to discover any interesting endpoint.

{: .box-note}
**Note:** Add `10.10.11.64` in `/etc/hosts` file and named it to `nocturnal.htb`.

![image](/assets/img/webnocturnal.png){: .mx-auto.d-block :}

### Directory Enumeration

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

There is four endpoint discovered through `gobuster` where the interesting one is `admin.php`. Upon login to the web, user are able to upload files and download files that has been uploaded. There is no indication that this web are vulnerable through unrestricted file upload that will gain RCE or Reverse Shell.


## Gaining Access

Further look into the web reveal endpoint when downloading the file `/view.php?username=test1&file=sample.pdf`. The `username` parameter will be use as a point to enumerate user that available in this web application.

![image](/assets/img/burp1.png){: .mx-auto.d-block :}

### User Enumeration

{% highlight bash linenos %}
└─$ ffuf -w /usr/share/wordlists/seclists/Usernames/Names/names.txt -u 'http://nocturnal.htb/view.php?username=FUZZ&file=test.pdf' -H 'Cookie: PHPSESSID=olt58v8arrqqotc1ckci6q7qlq' -fs 2985


        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://nocturnal.htb/view.php?username=FUZZ&file=test.pdf
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Usernames/Names/names.txt
 :: Header           : Cookie: PHPSESSID=olt58v8arrqqotc1ckci6q7qlq
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 2985
________________________________________________

admin                   [Status: 200, Size: 3037, Words: 1174, Lines: 129, Duration: 21ms]
amanda                  [Status: 200, Size: 3113, Words: 1175, Lines: 129, Duration: 15ms]
tobias                  [Status: 200, Size: 3037, Words: 1174, Lines: 129, Duration: 17ms]
:: Progress: [10177/10177] :: Job [1/1] :: 2173 req/sec :: Duration: [0:00:08] :: Errors: 0 ::
{% endhighlight %}

Output from the user enumeration reveal that it is vulnerable to insecure direct object reference (IDOR) whereby when visiting each users will reveal what files that available for user to download. From those three users, only amanda has file called `privacy.odt` which reveal the message from nocturnal IT team that exposed amanda temporary password.

![image](/assets/img/idor.png){: .mx-auto.d-block :}
![image](/assets/img/expass.png){: .mx-auto.d-block :}

With the exposed credential, try to use it on ssh seems no luck. This leave us to login to web using those credential and luckly amanda able to access admin page. The admin page can review source code of each pages available in the web and the most interesting part are on `admin.php`.

![image](/assets/img/blacklist.png){: .mx-auto.d-block :}

In the admin.php source code, there is some minimal blacklists that is implement to avoid user from doing the command injection. It is because from user input the `password` would be directly pass in the command as part of one full command to make a backup zip files. But this can be easily bypass with `\r\n` where this will be the point to split the command to execute another one and `\t` as substitue to space that has been blacklists.

![image](/assets/img/code1.png){: .mx-auto.d-block :}
![image](/assets/img/burp2.png){: .mx-auto.d-block :}

When exploring through the directory, it is reveal that there is another interesting file called `./nocturnal_database/nocturnal_database.db` which exposed all the users and password hashes through encoded the file to base64 to exfiltrate the database. On all of the three password hashes, tobias are able to be decrypted and use it to gain access through ssh.

![image](/assets/img/burp3.png){: .mx-auto.d-block :}

| User | Password |
| :------ |:---------------- |
| admin | Six |
| amanda | Eleven |
| tobias | Eight |

![image](/assets/img/tobiasssh.png){: .mx-auto.d-block :}


## Privilege Escalation To Root

