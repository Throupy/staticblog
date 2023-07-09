---
title: TryHackMe MD2PDF Writeup
date: 2023-07-09T12:07:32.595Z
tags:
  - writeup
description: My writeup for the TryHackMe machine "MD2PDF". Difficulty - very easy.
---
This machine is probably one of the easiest machines on the TryHackMe platform.

## Reconnaissance

```shell
$ nmap -sV 10.10.170.93
```

```shell
Starting Nmap 7.92 ( https://nmap.org ) at 2023-07-09 07:58 EDT
WARNING: Service 10.10.170.93:5000 had already soft-matched rtsp, but now soft-matched sip; ignoring second value
WARNING: Service 10.10.170.93:80 had already soft-matched rtsp, but now soft-matched sip; ignoring second value
Nmap scan report for 10.10.170.93
Host is up (0.023s latency).
Not shown: 997 closed tcp ports (conn-refused)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
80/tcp   open  rtsp
5000/tcp open  rtsp
```

There was a web server running on HTTP80, so I took a look in a browser and ran a `dirsearch` in the background to discover directories and routes.

```shell
$ dirsearch -u http://10.10.170.93
....
[08:03:38] Starting: 
[08:03:45] 403 -  166B  - /admin
```

There was a /admin route on the web server, when I tried to navigate to it, I got the following:

![](/img/admin-forbidden.png)

It said that the admin route can only be accessed internally. Given that this is a machine with an `XSS` tag on it, I assumed that there must be some way to get the contents via the markdown input section on the main page (below)

![](/img/mainpage3.png)

My first thought was to use HTML script tags with jQuery to get the contents of the page, and send it to my own hosted web server, but after looking online for XSS with markdown, I figured that using an `iframe` would probably just be easier. Below is a screenshot of the payload that I used.

![](/img/payload.png)

The payload created an Iframe in the page, which is a way of embedded webpages within webpages. Given that the admin panel can only be accessed internally, I could access it via the markdown panel as the context was internal. The `src` attribute on the iframe element is what specifies the webpage address to load into the parent webpage. When I submitted the payload, this is what I got as a result.

![](/img/flag.png)

And that's it... the flag already. This machine was very easy and I was a little dissapointed!