---
title: TryHackMe The Marketplace Writeup
date: 2023-07-05T13:34:41.636Z
description: My solution for the TryHackMe Room "The marketplace"
---
T﻿his room is rated at medium difficulty and covers some interesting topics such as XSS (cross-site scripting), SQL Injection, and a couple of methods for privilege escalation. For me, the victim machine is at address `10.10.15.67`

## R﻿econnaissance

S﻿tarting with an `nmap `scan to find running services and open ports.

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

T﻿he website seems to be some sort of online marketplace (surprise, surprise). Poking around on the website uncovers the ability to log in or sign up to create an account. After signing up and logging in, there were some new option available to me: the ability to create a listing, and to view messages for my account. As well as this, there was a session cookie created with the name `token`. These session cookies are used by the browser in order to tell what user is logged in, in order to show them the correct content on the page.





T﻿BC