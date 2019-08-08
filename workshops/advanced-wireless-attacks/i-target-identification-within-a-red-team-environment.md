---
title: I. Target Identification Within A Red Team Environment
layout: workshop
---

# Table of Contents

   * [Workshop Overview](http://solstice.sh/workshops/advanced-wireless-attacks/)
   * ***[I. Target Identification Within A Red Team Environment](http://solstice.sh/workshops/advanced-wireless-attacks/i-target-identification-within-a-red-team-environment/)***
   * [II. Attacking and Gaining Entry To WPA2-EAP Wireless Networks](http://solstice.sh/workshops/advanced-wireless-attacks/ii-attacking-and-gaining-entry-to-wpa2-eap-wireless-networks/)
   * [III. EAP Downgrade Attacks](http://solstice.sh/workshops/advanced-wireless-attacks/ii-eap-downgrade-attacks/)
   * [IV. Wireless Man-In-The-Middle Attacks](http://solstice.sh/workshops/advanced-wireless-attacks/iii-wireless-man-in-the-middle-attacks/)
   * [V. SMB Relays and LLMNR/NBT-NS Poisoning](http://solstice.sh/workshops/advanced-wireless-attacks/iv-smb-relays-and-llmnr-nbt-ns-poisoning/)
   * [VI. Firewall And NAC Evasion Using Indirect Wireless Pivots](http://solstice.sh/workshops/advanced-wireless-attacks/v-firewall-and-nac-evasion-using-indirect-wireless-pivots/)

---

# Chapter Overview

Like any form of hacking, well executed wireless attacks begin with well-executed recon. During a typical wireless assessment, the client will give you a tightly defined scope. You’ll be given a set of ESSIDs that you are allowed to engage, and you may even be limited to a specific set of BSSIDs.

During red team assessments, the scope is more loosely defined. The client will typically hire your company to compromise the security infrastructure of their entire organization, with a very loose set of restrictions dictating what you can and can’t do. To understand the implications of this, let’s think about a hypothetical example scenario. 

Evil Corp has requested that your firm perform a full scope red team assessment of their infrastructure over a five week period. They have 57 offices across the US and Europe, most of which have some form of wireless network. The client seems particularly interested in wireless as a potential attack vector.

You and your team arrive at one of the client sites and perform a site-survey using airodump-ng, and see the output shown below.

![evil twin attack](http://solstice.sh/images/workshops/awae/i/image-1.png)

None of the networks within range have an ESSID that conclusively ties them to Evil Corp. Many of them do not broadcast ESSIDs, and as such lack any identification at all.

To make matters worse, the branch location your team is scoping out is immediately adjacent to a major bank, a police station, and numerous small retail outlets. It is very likely that many of the access points that you see in your airodump-ng output belong to one of these third parties. 

![evil twin attack](http://solstice.sh/images/workshops/awae/i/image-2.png)

During some engagements, you may be able to reach out to the client at this point and ask for additional verification. More likely, however, you’ll have to identify in-scope targets yourself. Let’s talk about how to do this.

# Scoping A Wireless Assessment: Red Team Style

We have four primary techniques at our disposal that we can use to identify in-scope wireless targets:

 - Linguistic Inference
 - Sequential BSSID patterns
 - Geographic cross-referencing
 - OUI Prefixes

Let’s talk about each of these techniques in detail.

## Linguistic Inference

Using Linguistic Inference during wireless recon is the process of identifying access points with ESSIDs that are linguistically similar to words or phrases that the client uses to identify itself. For example, if you are looking for access points owned by Evil Corp and see a network named “EvilCorp-guest”, it is very likely (although not certain) that this network is in-scope. Similarly, if you’re pentesting Evil Corp but see an ESSID named “US Department of Fear,” you should probably avoid it.

## Sequential BSSID Patterns

In our airodump-ng output, you may notice groups of BSSIDs that increment sequentially. For example:

![evil twin attack](http://solstice.sh/images/workshops/awae/i/image-4.png)

When you see a group of APs with BSSIDs that increment sequentially, as shown above, it usually means that they are part of the same network. If we identify one of the BSSIDs as in-scope, we can usually assume that the same is true for the rest of the BSSIDs in the sequence.

![evil twin attack](http://solstice.sh/images/workshops/awae/i/image-5.png)

## OUI Prefixes

The first three octets in a mac address identify the manufacture of the device. If we discover evidence that the client has a contract with certain hardware manufactures, we focus our attention on APs with OUI prefixes that correspond to these brands.

# Using Geographic Cross-Referencing To Identify In-Scope Access Points

The most powerful technique we can leverage is geographic cross-referencing. We mentioned that Evil Corp has 57 offices worldwide. If we see the same ESSIDs appear at two Evil Corp locations, and no other 3rd party is present at both of these locations as well, it is safe to conclude that the ESSID is used by Evil Corp.

We’ll apply these principles to identify in-scope access points in our airodump-ng output from the last section. Before we continue, we should attempt to decloak any wireless access points that have hidden ESSIDs. We do this by very briefly deauthenticating one or more clients from each of the hidden networks. Our running airodump-ng session will then sniff the ESSID of the affected access point as the client reassociates. Tools such as Kismet will do this automatically, although on sensitive engagements it’s preferable to do this manually in a highly controlled fashion.

To perform the deauthentication attack, we use the following command:

	root@localhost:~# aireplay-ng -b <name of bssid here> -c <mac address of client (optional)> <interface name>

If successful, the ESSID of the affected access point will appear in our airodump-ng output.

![evil twin attack](http://solstice.sh/images/workshops/awae/i/image-6.png)

We have now identified six unique ESSIDs present at the client site. We can cross reference these ESSIDs with the results of similar site surveys performed at nearby client sites. Suppose we know of another Evil Corp branch office 30 miles away. After driving to the secondary location, we discover that none of the third-party entities located at the first branch office are present. Suppose that after decloaking hidden networks, we see that the following access points are within range. 

![evil twin attack](http://solstice.sh/images/workshops/awae/i/image-7.png)

Out of the ESSIDs shown above, the ones that are highlighted in red were also found at the first Evil Corp location that we visited. Given the lack of common third-parties present at each of the sites, this is very strong evidence that these ESSIDs are used by Evil Corp, and are therefore in-scope.

# Expanding The Scope By Identifying Sequential BSSIDs 

Let’s return to the first client site. Notice that the BSSIDs outlined in red increment sequentially. As previously mentioned, this usually occurs when the APs are part of the same network. We know that EC7293 is in-scope (we confirmed this using geographic cross-referencing). Given that the access points serving EC7293 and ECwnet1 are part of the same group of sequentially incrementing BSSIDs, we can conclude they are both parts of the same network. Therefore, it follows that ECwnet1 is in-scope as well.

![evil twin attack](http://solstice.sh/images/workshops/awae/i/image-8.png)

We’ve mapped our in-scope attack surface. Our targets will be the following access points:

![evil twin attack](http://solstice.sh/images/workshops/awae/i/image-9.png)

---

### Next chapter: *[II. Attacking and Gaining Entry To WPA2-EAP Wireless Networks](http://solstice.sh/workshops/advanced-wireless-attacks/ii-attacking-and-gaining-entry-to-wpa2-eap-wireless-networks/)*

---
