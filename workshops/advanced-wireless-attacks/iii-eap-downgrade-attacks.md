---
title: III. EAP Downgrade Attacks
layout: workshop
---

# Table of Contents

   * [Workshop Overview](http://solstice.sh/workshops/advanced-wireless-attacks/)
   * [I. Target Identification Within A Red Team Environment](http://solstice.sh/workshops/advanced-wireless-attacks/i-target-identification-within-a-red-team-environment/)
   * [II. Attacking and Gaining Entry To WPA2-EAP Wireless Networks](http://solstice.sh/workshops/advanced-wireless-attacks/ii-attacking-and-gaining-entry-to-wpa2-eap-wireless-networks/)
   * ***[III. EAP Downgrade Attacks](http://solstice.sh/workshops/advanced-wireless-attacks/ii-eap-downgrade-attacks/)***
   * [IV. Wireless Man-In-The-Middle Attacks](http://solstice.sh/workshops/advanced-wireless-attacks/iii-wireless-man-in-the-middle-attacks/)
   * [V. SMB Relays and LLMNR/NBT-NS Poisoning](http://solstice.sh/workshops/advanced-wireless-attacks/iv-smb-relays-and-llmnr-nbt-ns-poisoning/)
   * [VI. Firewall And NAC Evasion Using Indirect Wireless Pivots](http://solstice.sh/workshops/advanced-wireless-attacks/v-firewall-and-nac-evasion-using-indirect-wireless-pivots/)

---

# Chapter Overview

In this section we'll go over some additional attacks against EAP that can be used to coerce wireless clients into using insecure EAP methods. We'll first discuss how the EAP Negotiation process works and how it is implemented by hostapd. We'll then discuss both generic and GTC downgrade attacks against EAP.

# Wireless Theory: EAP Negotiation and the EAP User File

In order to talk about how EAP Downgrade attacks work, we first need to understand how hostapd handles user accounting and EAP Negotiation. Extensible Authentication Protocol (EAP) is really more of a framework rather than a specific authentication mechanism. It provides a standardized set of functions and rules that dictate how authentication should take place, but leaves the specific details to individual implementations known as EAP Methods. 

Hostapd keeps track of which EAP methods should be supported it's EAP User file, which is also used to manage user account information. You can think of the EAP User file as an account database, since this is the function it serves despite being implemented as a text file.

In order to talk about how EAP Downgrade attacks work, we first need to talk about hostapd's EAP User file, which is used by hostapd to manage user account information. You can think of it as an account database, since this is the function it serves despite being implemented as a text file.

Each line in the EAP User file corresponds to a user account, and contains the following attributes:

- a user's identity
- a list of EAP methods that the user is permitted to use
- an optional password. 

_Note: an NT password hash can be used in leui of a password if MSCHAP or MSCHAPv2 is used for authentication.  The NT password hash is the 16-byte MD4 hash of the unicode representation of the password._

When a user attempts to authenticate with hostapd using EAP, hostapd searches for the user in the EAP User file by identity. When a matching line in the file is found, hostapd iterates through the list of EAP methods for the user from left to right. At each iteration, hostapd asks the client device to authenticate using the currently selected EAP method. Each time hostapd suggests an EAP method to the client, the client is given the choice to accept or deny it. Hostapd will continue to suggest EAP methods until client agrees to one of them, or the list of allowed EAP methods is exhausted (in the latter case, authentication fails). This process is known as EAP negotiation.

 his process, which is known as EAP Negotiation, continues until the client agrees to an EAP method suggested by the server or the list of allowed EAP methods is exhausted (in the latter case, authentication fails).

# Wireless Theory: EAP Downgrade Attacks

As shown in in the example below, which contains a line from a default hostapd EAP User file, the list of supported EAP methods is ordered from strongest to weakest.  This is done to ensure that client devices use the strongest EAP method available.

```
"t"	TTLS-PAP,TTLS-CHAP,TTLS-MSCHAP,MSCHAPV2,MD5,GTC,TTLS,TTLS-MSCHAPV2	"t"	[2]
```

To implement a generic EAP downgrade attack, we simply reverse the order of this list. This causes hostapd to suggest the weakest EAP methods first, and often results in client devices authenticating using plaintext methods such as PAP or EAP-GTC. 

```
"t"	GTC,TTLS-PAP,MD5,TTLS-CHAP,TTLS-MSCHAP,MSCHAPV2,TTLS-MSCHAPV2,TTLS	"t"	[2]
```

Note that this also slows down the EAP negotiation process, since many client devices won't accept the first few EAP methods suggested by hostapd. This may make it more difficult to successfully execute a rogue EAP attack, since the client device may abort the authentication attempt and connect to one of its friendly access points instead. 

# GTC Downgrade Attacks

EAP-GTC is an EAP method created by Microsoft and Cisco to support the use of hardware tokens and one time passwords with MSCHAPv1. It's similar to PEAPv0 EAP-MSCHAPv2, but does not have a peer challenge. Instead, passwords are sent to the access point in plaintext.

Many client devices fail to state what kind of password they're asking for when prompting users for credentials. Instead, the user is presented with a generic login prompt. This means that if we can force a client device to use GTC to prompt a user for a one time password, the user is likely to believe that the device is asking for network logon credentials.

For the purposes of this writeup, we're going to seperate GTC Downgrade attacks into two categories:

- Full EAP Downgrade attacks
- GTC Downgrade attacks

In a Full EAP Downgrade attack, we reverse the order of the target user's allowed EAP methods in hostapd's EAP user file, making sure that GTC is suggested first. The advantage of this style of attack is that it allows some flexibility to target clients that do not support GTC.

In a GTC Downgrade attack, we explicitly instruct client devices to authenticate using EAP-GTC as shown in the following EAP User entry:

```
* PEAP [ver=1]
"t" GTC "t" [2]
```

# Lab Exercise: EAP Downgrade attack

Use eaphammer to create a WPA/2-EAP access point with the `--eap-downgrade full` flag:

```bash
./eaphammer -i wlan0 -e exampleWiFi --auth wpa-eap --creds --eap-downgrade full
```

Then, attempt to connect to the access point using your Windows 8 VM, Windows 10 VM, your host operating system, and as many different mobile devices as you can get your hands on. Compare what occurs when each device attempts to authenticate with the access point.

# Lab Exercise: GTC Downgrade attack

Use eaphammer to create a WPA/2-EAP access point with the `--eap-downgrade gtc` flag:

```bash
./eaphammer -i wlan0 -e exampleWiFi --auth wpa-eap --creds --eap-downgrade gtc
```

Then attempt to connect to the access point using your Windows 8 VM, Windows 10 VM, your host operating system, and as many different mobile devices as you can get your hands on. Compare what occurs when each device attempts to authenticate with the access point.

# Lab Exercise: Manual EAP Downgrade attack

Use eaphammer to create a WPA/2-EAP access point with the `--eap-downgrade manual` and `--eap-methods` flags:

```bash
./eaphammer -i wlan0 -e exampleWiFi --auth wpa-eap --creds --eap-downgrade manual --eap-methods EAP-MD5 TTLS-MSCHAPv2
```

Attempt to connect to the access point using a variety of different devices as in the previous lab exercises. Make sure to play with the `--eap-methods` by trying different EAP methods.

---

### Next chapter: *[IV. Wireless Man-In-The-Middle Attacks](http://solstice.sh/workshops/advanced-wireless-attacks/iv-wireless-man-in-the-middle-attacks/)*

---
