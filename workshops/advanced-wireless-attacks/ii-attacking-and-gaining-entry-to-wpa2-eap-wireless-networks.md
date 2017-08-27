---
title: II. Attacking And Gaining Entry To WPA2-EAP Wireless Networks
layout: workshop
---

# Table of Contents

   * [Workshop Overview]({{ site.baseurl }}workshops/advanced-wireless-attacks/)
   * [I. Target Identification Within A Red Team Environment]({{ site.baseurl }}workshops/advanced-wireless-attacks/i-target-identification-within-a-red-team-environment/)
   * ***[II. Attacking and Gaining Entry To WPA2-EAP Wireless Networks]({{ site.baseurl }}workshops/advanced-wireless-attacks/ii-attacking-and-gaining-entry-to-wpa2-eap-wireless-networks/)***
   * [III. Wireless Man-In-The-Middle Attacks]({{ site.baseurl }}workshops/advanced-wireless-attacks/iii-wireless-man-in-the-middle-attacks/)
   * [IV. SMB Relays and LLMNR/NBT-NS Poisoning]({{ site.baseurl }}workshops/advanced-wireless-attacks/iv-smb-relays-and-llmnr-nbt-ns-poisoning/)
   * [V. Firewall And NAC Evasion Using Indirect Wireless Pivots]({{ site.baseurl }}workshops/advanced-wireless-attacks/v-firewall-and-nac-evasion-using-indirect-wireless-pivots/)

---

# Chapter Overview

Rogue access point attacks are the bread and butter of modern wireless penetration tests. They can be used to perform stealthy man-in-the-middle attacks, steal RADIUS credentials, and trick users into interacting with malicious captive portals. Penetration testers can even use them for traditional functions such as deriving WEP keys and capturing WPA handshakes [1]. Best of all, they are often most effective when used out of range of the target network. For this workshop, we will focus primarily on using Evil Twin attacks.

# Wireless Theory: Evil Twin Attacks

An Evil Twin is a wireless attack that works by impersonating a legitimate access point. The 802.11 protocol allows clients to roam freely from access point to access point. Additionally, most wireless implementations do not require mutual authentication between the access point and the wireless client. This means that wireless clients must rely exclusively on the following attributes to identify access points:

1. **BSSID** – The access point’s Basic Service Set identifier, which refers to the access point and every client that is associated with it. Usually, the access point's MAC address is used to derive the BSSID.
2. **ESSID** – The access point’s Extended Service Set identifier, known colloquially as the AP’s “network name.” An Extended Service Set (ESS) is a collection of Basic Service Sets connected using a common Distribution System (DS).
3. **Channel** – The operating channel of the access point.

To execute the attack, the attacker creates an access point using the same ESSID and channel as a legitimate AP on the target network. So long as the malicious access point has a more powerful signal strength than the legitimate AP, all devices connected to the target AP will drop and connect to the attacker.

# Wireless Theory: WPA2-EAP Networks

Now let’s talk about WPA2-EAP networks. The most commonly used EAP implementations are EAP-PEAP and EAP-TTLS. Since they’re very similar to one another from a technical standpoint, we’ll be focusing primarily on EAP-PEAP. However, the techniques learned in this workshop can be applied to both.

The EAP-PEAP authentication process is an exchange that takes place between three parties: the wireless client (specifically, software running on the wireless client), the access point, and the authentication server. We refer to the wireless client as the supplicant and the access point as the authenticator [2]. 
 
Logically, authentication takes place between the supplicant and the authentication server. When a client device attempts to connect to the network, the authentication server presents the supplicant with an x.509 certificate. If the client device accepts the certificate, a secure encrypted tunnel is established between the authentication server and the supplicant. The authentication attempt is then performed through the encrypted tunnel. If the authentication attempt succeeds, the client device is permitted to associate with the target network [2][3].

![evil twin attack]({{ site.baseurl }}images/workshops/awae/ii/image-1.png)

Without the use of the secure tunnel to protect the authentication process, an attacker could sniff the challenge and response then derive the password offline. In fact, legacy implementations of EAP, such as EAP-MD5, are susceptible to this kind of attack. However, the use of a secure tunnel prevents us from using passive techniques to steal credentials for the target network [2][3].

![evil twin attack]({{ site.baseurl }}images/workshops/awae/ii/image-4.png)

Although we can conceptualize the EAP-PEAP authentication process as an exchange between the supplicant and the authentication server, the protocol’s implementation is a bit more complicated. All communication between the supplicant and the authentication server is relayed by the authenticator (the access point). The supplicant and the authenticator communicate using a Layer 2 protocol such as IEEE 802.11X, and the authenticator communicates with the authentication server using RADIUS, which is a Layer 7 protocol. Strictly speaking, the authentication server and supplicant do not actually communicate directly to one another at all [2][3].

![evil twin attack]({{ site.baseurl }}images/workshops/awae/ii/image-2.png)


As you can imagine, this architecture creates numerous opportunities for abuse once an attacker can access the network. However, for now, we’re going to focus on abusing this authentication process to gain access to the network in the first place. Let’s revisit the EAP-PEAP authentication process, but this time in further detail.

![evil twin attack]({{ site.baseurl }}images/workshops/awae/ii/image-3.png)

The diagram above illustrates the EAP-PEAP/EAP-TTLS authentication process in full detail. When the supplicant associates, it sends an EAPOL-Start to the authenticator. The authenticator then sends an EAP-Request Identity to the supplicant. The supplicant responds with its identity, which is forwarded to the authentication server. The authentication server and supplicant then setup a secure SSL/TLS tunnel through the authenticator, and the authentication process takes place through this tunnel [2].

Until the tunnel is established, the authenticator is essentially acting as an open access point. Even though the authentication process occurs through a secure tunnel, the encrypted packets are still being sent over open wireless. The WPA2 doesn’t kick in until the authentication process is complete. Open wireless networks are vulnerable to Evil Twin attacks because there is no way for the wireless client to verify the identity of the access point to which it is connecting. Similarly, EAP-PEAP and EAP-TTLS networks are vulnerable to Evil Twin attacks because there is no way to verify the identity of the authenticator [4].

In theory, the certificate presented to the supplicant by the authentication server could be used to verify the identity of the authentication server. However, this is only true so long as the supplicant does not accept invalid certificates. Many supplicants do not perform proper certificate validation. Many other are configured by users or administrators to accept untrusted certificates automatically. Even if the supplicant is configured correctly, the onus is still placed on the user to decline the connection when presented with an invalid certificate [5].

To compromise EAP-PEAP, the attacker first performs an Evil Twin attack against the target access point (which is serving as the authenticator). When a client connects to the rogue access point, it begins an EAP exchange with the attacker’s authenticator and authentication server. If the supplicant accepts the attacker’s certificate, a secure tunnel is established between the attacker’s authentication server and the supplicant. The supplicant then completes the authentication process with the attacker, and the attacker uses the supplicant’s challenge and response to derive the victim’s password [4][5].

# Evil Twin Attack Against WPA2-EAP (PEAP)

The first phase of this attack will be to create an Evil Twin using eaphammer. Traditionally, this attack is executed using a tool called hostapd-wpe. Although hostapd-wpe is a powerful tool in its own right, it can be quite cumbersome to use and configure. Eaphammer provides an easy to use command line interface to hostapd-wpe and automates its traditionally time-consuming configuration process.

We’ll begin by creating a self-signed certificate using eaphammer’s --cert-wizard flag.

	./eaphammer --cert-wizard

The Cert Wizard routine will walk you through the creation of a self-signed x.509 certificate automatically. You will be prompted to enter values for a series of attributes that will be used to create your cert. 

It's best to choose values for your self-signed certificate that are believable within the context of your target organization. Since we’re attacking Evil Corp, the following examples values would be good choices:

1.	Country – US
2.	State – Utah
3.	Locale – Salt Lake City
4.	Organization – Evil Corp
5.	Email – admin@evilcorp.com
6.	CN – admin@evilcorp.com

When the Cert Wizard routine finishes, you should see output similar to what is shown in the screenshot below.

![evil twin attack]({{ site.baseurl }}images/workshops/awae/ii/image-5.png)

Once we have created a believable certificate, we can proceed to launch an Evil Twin attack against one of the target access points discovered in the last section. Let’s use eaphammer to perform an Evil Twin attack against the access point with BSSID 1c:7e:e5:97:79:b1.

./eaphammer --bssid 1C:7E:E5:97:79:B1 --essid ECwnet1 --channel 2 --wpa 2 --auth peap --interface wlan0 --creds

Provided you can overpower the signal strength of the target access point, clients will begin to disconnect from the target network and connect to your access point. Unless the affected client devices are configured to reject invalid certificates, the victims of the attack will be presented with a message similar to the one below. 

![evil twin attack]({{ site.baseurl }}images/workshops/awae/ii/image-7.png)

Fortunately, it’s usually possible to find at least one enterprise employee who will blindly accept your certificate. It’s also common to encounter devices that are configured to accept invalid certificates automatically. In either case, you’ll soon see usernames, challenges, and responses shown in your terminal as shown below.

![evil twin attack]({{ site.baseurl }}images/workshops/awae/ii/image-6.png)

This data can be passed to asleap to obtain a valid set of RADIUS credentials.

	asleap –C <challenge> -R <response> -W <wordlist>

Congrats. You have your first set of RADIUS creds.

![evil twin attack]({{ site.baseurl }}images/workshops/awae/ii/image-8.png)

# Lab Exercise: Evil Twin Attack Against WPA2-PEAP

For this lab exercise, you will practice stealing RADIUS credentials by performing an Evil Twin attack against a WPA2-EAP network.

1.	Using your wireless router:
	1.	Create a WPA2-EAP network with the EAP type set to PEAP or TTLS. Make sure to set the EAP password to “2muchswagg” without the quotes.
2.	From your Windows AD Victim VM:
	1.	Connect to the WPA2-EAP network using your secondary external wireless adapter.
3.	From your Kali Linux VM:
	1.	Use airodump-ng to identify your newly created WPA2-EAP network
	2.	Use eaphammer to generate a believable self-signed certificate
	3.	Use eaphammer to capture an EAP Challenge and Response by performing an Evil Twin attack against the WPA2-EAP network
	4.	Use asleap to obtain a set of RADIUS credentials from the Challenge and Response captured in step 3b. Make sure to use the rockyou wordlist, which is located at /usr/share/wordlists/rockyou.txt. 

---

### Next chapter: *[III. Wireless Man-In-The-Middle Attacks]({{ site.baseurl }}workshops/advanced-wireless-attacks/iii-wireless-man-in-the-middle-attacks/)*

---
