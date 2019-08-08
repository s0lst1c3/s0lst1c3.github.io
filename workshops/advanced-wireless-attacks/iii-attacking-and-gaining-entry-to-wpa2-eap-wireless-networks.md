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

In this section we'll go over some additional advanced attacks that can be leveraged against networks that use WPA/2-EAP. The first of these attacks is the GTC Downgrade attack, which can be used to trick client devices into surrendering plaintext credentials. The second of these attacks is the EAP Relay attack, which can be used to to relay inner MSHCHAPv2 to authenticate with PEAP networks without cracking passwords (think NTLM relay attacks for WiFi networks).

# Wireless Theory: EAP Negotiation and the EAP User File

In order to talk about how EAP Downgrade attacks work, we first need to talk about hostapd's EAP User file, which is used by hostapd to manage user account information. Each line in this file contains the following attributes:

- the user's identity
- a list of EAP methods that the user is permitted to use
- an optional password. 

!!!Note: an NT password hash can be used in leui of a password if MSCHAP or MSCHAPv2 is used for authentication.
!!!NT password hash: 16-byte MD4 hash of the unicode representation of the password

When a user attempts to authenticate with hostapd using EAP, hostapd looks up the user in the EAP User file by their identity. When a matching line in the file is found, hostapd iterates through the list of EAP methods for that user from left to right. At each iteration, hostapd asks the client device to authenticate using the currently selected EAP method. The client device can then either accept or deny the suggested EAP method. This process, which is known as EAP Negotiation, continues until the client agrees to an EAP method suggested by the server or the list of allowed EAP methods is exhausted (in the latter case, authentication fails).

As shown in FIGURE X below, which contains a line from a default hostapd EAP User file, the list of supported EAP methods is ordered from strongest to weakest.  This is done to ensure that client devices use the strongest EAP method available.

```snippet here```

# Wireless Theory: EAP Downgrade Attacks

To implement a generic EAP downgrade attack, we reverse the order of these EAP methods as shown in FIGFURE X below. This causes hostapd to suggest the weakest EAP methods first, and often results in client devices authenticating using plaintext methods such as PAP or EAP-GTC. 

# GTC Downgrade Attacks

EAP-GTC is an EAP method created by Microsoft and Cisco to support the use of hardware tokens and one time passwords with MSCHAPv1. It's similar to PEAPv0 EAP-MSCHAPv2, but does not have a ppeer challenge. Instead, password are sent to the access point in plaintext.

Many client devices don't state what kind of password they're asking for when prompting the user for credentials. Instead, the user is presented with a generic login prompt that asks for a username and password. This means that if we can somehow prompt the user for a one time password using GTC, they're likely to believe that they're being prompted for their network logon credentials.

For the purposes of this writeup, we're going to seperate GTC Downgrade attacks into two categories:

- Suggested GTC Downgrade attacks
- Forced GTC Downgrade attacks

In a Suggested GTC Downgrade attack, we reverse the order of the target user's allowed EAP methods in hostapd's EAP user file, making sure that GTC is suggested first. EAPHammer is configured to perform this style of attack by default, as it allows some flexibility to target clients that do not support GTC.

```snippet here```

In a Forced GTC Downgrade attack, we explicitly instruct client devices to authenticate using EAP-GTC as shown in the following EAP User entry:


```snippet here```

---

### Next chapter: *[III. Wireless Man-In-The-Middle Attacks](http://solstice.sh/workshops/advanced-wireless-attacks/iii-wireless-man-in-the-middle-attacks/)*

---
