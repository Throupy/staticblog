---
title: Kerberos Authentication
date: 2023-07-02T18:56:23.473Z
description: A look at how Kerberos authentication works on Windows networks.
  First blog post so testing how CMS functions and if markdown working as
  expected.
---
### Key Terms

Kerberos authentication is very complicated, here is a list of terms and definitions to refer back to.

* `KDC` - Key Distribution Centre - A service which handles the distribution of Kerberos tickets on a network. This service is usually installed on the DC
* `TGT` - Ticket granting ticket - like a security badge, allows a user to request tickets from the KDC without having to authenticate every time.
* `Session Key` - A session key (believe it or not!) used by the user to generate further requests to send to the KDC
* `krbtgt` - The default account for the Kerberos service
* `TGS` - Ticket granting service - Tickets that allow connection to a specific service, and only that service.
* `SPN` - Service Principle Name - An object which indicates the service and server name a user intends to access
* `Service Session Key` - A session key, but for communication with a service rather than the KDC
* `Service Owner Hash` - The password hash of the service owner, which is the account / machine under which a service runs
* `PAC` - Privilege Attribute Certificate - holds all of the user’s information, it is sent along with the TGT to the KDC to be signed and to validate the user

### Step 1

* The user sends their username and a timestamp, encrypted using a key derived from their password, to the KDC (Key Distribution Centre), a service usually installed on the domain controller, which is in charge of creating Kerberos tickets on a network.

  * `Kerberos Ticket` - A certificate of authentication issued by the KDC.
* The KDC will send back a TGT (Ticket granting ticket), allowing the user to request tickets to access specific services without passing their credentials to the KDC again

  * You can think of the TGT like a security badge, once you have it, the KDC will not ask to see any identification or try and authenticate you.

  Along with the TGT, a session key will be given to the user, which must be used to generate the requests that follow.
* Note: the TGT is encrypted using the `krbtgt` account’s password hash, so the user can’t access its contents. The encrypted TGT contains the session key so the KDC does not need to track all sessions, as it can decrypt the TGT and retrieve the session key if needed.

  * `krbtgt` - the account for the Kerberos service on the domain controller

    ![](/img/step1.png)

  ### Step 2
* When users want to connect to a network service (such as a share, website or database), they will use their TGT to ask the KDC for a TGS (ticket granting service). TGS are tickets that allow connection only to the specific service for which they are created. To request a TGS, the user once again send their username and timestamp, but encrypted using the session key this time, along with their TGT and a SPN (Service principle name)

  * `SPN` - Indicates the service and server name you intend to access
* The KDC responds with a TGS and a `service session key`**,** which will be needed to authenticate to the service that is to be accessed. The TGS is encrypted using the `service owner hash.` The service owner is the user / account under which the service runs. The TGS contains a copy of the service session key on its encrypted contents so that the service owner can access it by decrypting the TGS

![](/img/step2.png)

### Step 3

* The user can then send the TGS to the service they want to access. The service will use its configured account’s password hash to decrypt the TGS and validate the service session key

![](/img/step3.png)