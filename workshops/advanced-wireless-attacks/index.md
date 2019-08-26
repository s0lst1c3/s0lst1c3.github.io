---
title: Advanced Wireless Attacks Against Enterprise Networks (AWAE) (v3.0.1)
layout: page
---

# Table of Contents

   * ***[Workshop Overview](http://solstice.sh/workshops/advanced-wireless-attacks/)***
   * [I. Target Identification Within A Red Team Environment](http://solstice.sh/workshops/advanced-wireless-attacks/i-target-identification-within-a-red-team-environment/)
   * [II. Attacking and Gaining Entry To WPA2-EAP Wireless Networks](http://solstice.sh/workshops/advanced-wireless-attacks/ii-attacking-and-gaining-entry-to-wpa2-eap-wireless-networks/)
   * [III. EAP Downgrade Attacks](http://solstice.sh/workshops/advanced-wireless-attacks/iii-eap-downgrade-attacks/)
   * [IV. Wireless Man-In-The-Middle Attacks](http://solstice.sh/workshops/advanced-wireless-attacks/iv-wireless-man-in-the-middle-attacks/)
   * [V. SMB Relays and LLMNR/NBT-NS Poisoning](http://solstice.sh/workshops/advanced-wireless-attacks/v-smb-relays-and-llmnr-nbt-ns-poisoning/)
   * [VI. Firewall And NAC Evasion Using Indirect Wireless Pivots](http://solstice.sh/workshops/advanced-wireless-attacks/vi-firewall-and-nac-evasion-using-indirect-wireless-pivots/)

---

# Preface

This is the online version of my Advanced Wireless Attacks workshop, which was first presented at DEF CON 25, BSides Las Vegas, and BSides SLC in 2017. It's seen some updates since then, and is likely to continue to be revised and expanded in the future. If you feel that this workshop is missing something, or have any feedback whatsoever, please don't hesitate to submit an issue on Github.

# Workshop Overview

In this workshop you will learn how to carry out sophisticated wireless attacks against corporate infrastructure. Topics of of interest include attacking and gaining access to WPA/2-EAP, bypassing network access controls, and exploring how wireless can be leveraged as a powerful means of lateral movement throughout an Active Directory environment.

Course Highlights include:

- Wireless Reconnaissance and Target Identification Within A Red Team Environment
- Attacking and Gaining Entry to WPA/2-EAP wireless networks
- SMB Relay Attacks and LLMNR/NBT-NS Poisoning
- Data Manipulation and Browser Exploitation Using Wireless MITM Attacks
- Downgrading Modern SSL/TLS Implementations Using Partial HSTS Bypasses
- EAP and GTC Downgrade Attacks
- Firewall and NAC Evasion Using Indirect Wireless Pivots

The material covered in this workshop is supplemented by hands-on lab exercises within the course's virtual lab environment. Instructions on setting up the virtual lab environment can be found in the Lab Setup Guide, which is included below. Feel free to reach out to the instructor at gryan[at]specterops[dot]io for guidance.


Prerequisites
-------------

A previous wireless security background is helpful but not required. You'll also need a laptop with at least 8 gb of RAM that is capable of running VirtualBox or VMWare. Additional hardware requirements can be in the Lab Setup Guide included below.

Course Materials
----------------

All course materials, including the lab setup guide, can be found at the following URL:

[Course Materials](https://drive.google.com/open?id=0BwFgM9oAhmd_c2JJaG1iUmhkZTg)

---

### First chapter: *[I. Target Identification Within A Red Team Environment](http://solstice.sh/workshops/advanced-wireless-attacks/i-target-identification-within-a-red-team-environment/)*

---

Changelog:
----------

## v3.0.1 - Mon Aug 26 2019

- Updated EAPHammer commandline syntax to work with latest version
- Content cleanup

## v3.0.0 

- Added additional course content: EAP Negotiation and Downgrade Attacks
- Presented at: _DEF CON 27_

## v2.x.x

- Shift from VirtualBox to VMWare for lab setup guide
- Shifted from manual lab installation to automated setup script
- Presented at: _DEF CON 26, BSides Las Vegas, BSides Chicago, BSides DC, 44con_

## v1.x.x

- Updates: Initial workshop  
- Presented at: _DEF CON 25, BSides Las Vegas, BSides Chicago, BSides DC, Hackfest_
