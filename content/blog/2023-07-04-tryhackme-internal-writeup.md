---
title: TryHackMe Internal Writeup
date: 2023-07-04T18:50:06.913Z
description: "A writeup of my solution for the TryHackMe 'Internal' machine "
---
Starting off with an \`nmap\` scan of the target which, for me, is the address \`10.10.234.17\`.

```
Starting Nmap 7.92 ( https://nmap.org ) at 2022-11-09 12:21 EST
Nmap scan report for 10.10.234.17
Host is up (0.025s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

From this we can deduce that an Apache web server is running on HTTP port 80, as well as an OpenSSH instance running on port 22. We can start checking out the Apache server by navigating to \`http://10.10.234.17\` in a browser.

When we navigate to this page we are presented with the default Apache landing page, as shown below.

![](/img/apache-landing-page.png)

We can use a tool called \`dirsearch\` to discover endpoints / routes on the webserver. The tool will try a series of commonly used endpoints from a wordlist.

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
[12:23:39] 301 -  311B  - /blog  ->  http://10.10.227.60/blog/              
[12:23:42] 200 -    4KB - /blog/wp-login.php                                
[12:23:42] 200 -   53KB - /blog/                                            
[12:23:46] 200 -   11KB - /index.html                                       
[12:23:47] 301 -  317B  - /javascript  ->  http://10.10.227.60/javascript/  
[12:23:52] 200 -   13KB - /phpmyadmin/doc/html/index.html                   
[12:23:52] 301 -  317B  - /phpmyadmin  ->  http://10.10.227.60/phpmyadmin/  
[12:23:54] 200 -   10KB - /phpmyadmin/index.php                             
[12:23:54] 200 -   10KB - /phpmyadmin/                                      
[12:23:56] 403 -  277B  - /server-status                                    
[12:23:56] 403 -  277B  - /server-status/                                   
[12:24:03] 200 -    4KB - /wordpress/wp-login.php
```

We notice that there is a \`/blog\` route that we can go to, let's look at that in the web browser.

![](/img/wordpress.png)

We are presented with a \`Wordpress\` site (can be deduced from \`Wappalyzer\` tool).

To be continued...