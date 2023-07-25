---
title: TryHackMe Anthem Writeup
date: 2023-07-25T19:40:20.984Z
description: My solution for the TryHackMe Anthem machine
---
## Reconnaissance

```
$ nmap -sV 10.10.29.150
Starting Nmap 7.92 ( https://nmap.org ) at 2023-07-25 15:35 EDT
Nmap scan report for 10.10.29.150
Host is up (0.024s latency).
Not shown: 998 filtered tcp ports (no-response)
PORT     STATE SERVICE
80/tcp   open  http
3389/tcp open  ms-wbt-server
```

From this we can see that there is a web server running on port 80, as well as an open RDP (remote desktop protocol) on port 3389.

We can answer the first few questions in the TryHackMe quiz for the machine:

* What port is for the web server? --> 80
* What port is for remote desktop service? --> 3389

Let's add the following line to `/etc/hosts` file.

```
...
10.10.29.150    anthem.thm
...
```

## Navigating to The Web Server

With the host file updated, we can navigate to `http://anthem.thm` . When navigating to the site we can see the following page.

![](/img/website.png)

The supporting content on Tryhackme points to "one of the pages web crawlers check for" and tells us to look for a password. This is likely `robots.txt`, so let's take a look in there. We can see there is a string `UmbracoIsTheBest! `at the top, which is likely the password. There are also files for `umbraco`, which is a CMS.

```
UmbracoIsTheBest!

# Use for all search robots
User-agent: *

# Define the directories not to crawl
Disallow: /bin/
Disallow: /config/
Disallow: /umbraco/
Disallow: /umbraco_client/
```

This allows us to answer the next couple of questions:

* What is a possible password in one of the pages web crawlers check for? --> UmbracoIsTheBest!
* What CMS is the website using? --> umbraco
* What is the domain of the website? --> anthem.com

## Administrator Details

One of the posts on the website is a tribute to the administrator, which contains a poem about him.

![](/img/poem.png)

A quick google search allows us to answer the following question:

* What's the name of the Administrator? --> Solomon Grundy

The other post on the site is by Jane Doe, notice that her email address is `JD@anthem.com`. That's first and last initial + @anthem.com. Now we can answer the last question:

* What is the administrator's email address? --> sg@anthem.com

## Finding Flags

Dissapointingly, the first two flags can be found try simply inspecting the source HTML of the 'cheers to our IT department' post. This is very underwhelming!

Flags 2 and 4 are `THM{G!T_G00D}` and `THM{AN0TH3R_M3TA}` respectively.

Flag 3 can be found on Jane Doe's profile page: `THM{L0L_WH0_D15}`

Flag 1 can be found in the 'we are hiring' page's meta tag: `THM{L0L_WH0_US3S_M3T4}`

Overall a pretty boring task!

## Accessing The Machine

After trying a few usernames (solomongrundy, sg@anthem.com, etc) with RDP, the credentials `sg:UmbracoIsTheBest!` finally got me access.

```shell
$ xfreerdp /v:10.10.29.150 /u:sg /p:UmbracoIsTheBest!
```

This will launch us to the user's desktop, where we can get the user flag: `THM{N00T_NO0T}`

Poking around the filestystem, there is a file in C:\Backup which, when given permissions, can be read and the password` ChangeMeBaby1MoreTime` is revealed. We can now log in as the administrator user in another RDP session

```shell
$ xfreerdp /v:10.10.29.150 /u:Administrator /p:ChangeMeBaby1MoreTime

```

And read the root flag as:`THM{Y0U_4R3_1337}`

## Final Thoughts

This machine was okay, even though it was rated at 'easy', the first two sections were very mundane. Viewing HTML source code for 4 flags is neither fun nor useful for learning. The last part was good, although a little short.