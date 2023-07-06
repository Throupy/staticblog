---
title: HackTheBox Support Machine Writeup
date: 2023-07-06T11:17:38.517Z
description: My solution for the HackTheBox machine called "Support"
---
## Setup

The target machine resided on the address `10.129.227.255` and I modified by hosts file as follows:

```
$ cat /etc/hosts
127.0.0.1       localhost
127.0.1.1       kali
10.129.227.255  support.htb
...

```

With this in place, I could then refer to `support.htb`, rather than supplying the IP address each time.

## Reconnaissance

I started with an `nmap` scan to discover running services and open ports.

```shell
$ nmap -sV -Pn support.htb
Not shown: 989 filtered ports
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-07-06 08:05:32Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: support.htb0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: support.htb0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows
```

From this I deduced that the target machine was running Windows with active directory, as hinted at by the LDAP service, amoungst others. There was also an SMB service present, so I started by enumerating the SMB file share.

## Enumerating SMB

I used the `smbclient` tool to enumerate all of the shares in the SMB service, and noticed an out-of-the-ordinary share called `support-tools.`

```
$ smbclient -L \\support.htb
    Sharename       Type      Comment
    ---------       ----      -------
    ADMIN$          Disk      Remote Admin
    C$              Disk      Default share
    IPC$            IPC       Remote IPC
    NETLOGON        Disk      Logon server share 
    support-tools   Disk      support staff tools
    SYSVOL          Disk      Logon server share 
```

I then examined the contents of the support-tools share.

```
$ smbclient --no-pass \\\\support.htb\\support-tools
smb: \> dir
  .
  ..
  7-ZipPortable_21.07.paf.exe
  npp.8.4.1.portable.x64.zip
  putty.exe
  SysInternalsSuite.zip
  UserInfo.exe.zip
  windirstat1_1_2_setup.exe
  WiresharkPortable64_3.6.5.paf.exe
```

The file UserInfo.exe.zip looked interesting to me, so I copied over to my kali machine, unzipped it and examined the contents. It seemed to be a windows executable program. I used mono to execute the program on my kali machine.

```
$ mono UserInfo.exe
Usage: UserInfo.exe [options] [commands]

Options:
  -v --verbose    Verbose output
 
Commands:
  find            Find a user
  user            Get information about a user
```

This program seemed to pull information about users, however I wasn't sure from what data source (database, active directory, etc). Active Directory seemed likely, given what I found in the reconnaissance phase.

Next, I opened the executable in `DNSpy`, which is a .NET debugger. This allowed me to look at the source code of the application.

![](/img/dnspy.png)

As I suspected, the application seems to be using LDAP to query active directory. Inside the `Userinfo.Services.LdapQuery` class I found the following function.

```csharp
  public LdapQuery()
  {
      string password = Protected.getPassword();
      this.entry = new DirectoryEntry("LDAP://support.htb", "support\\ldap", password);
      this.entry.AuthenticationType = AuthenticationTypes.Secure;
      this.ds = new DirectorySearcher(this.entry);
  }
```