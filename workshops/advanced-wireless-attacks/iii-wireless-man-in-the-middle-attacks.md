---
title: III. Wireless Man-In-The-Middle Attacks
layout: workshop
---

# Table of Contents

   * [Workshop Overview](http://solstice.sh/workshops/advanced-wireless-attacks/)
   * [I. Target Identification Within A Red Team Environment](http://solstice.sh/workshops/advanced-wireless-attacks/i-target-identification-within-a-red-team-environment/)
   * [II. Attacking and Gaining Entry To WPA2-EAP Wireless Networks](http://solstice.sh/workshops/advanced-wireless-attacks/ii-attacking-and-gaining-entry-to-wpa2-eap-wireless-networks/)
   * ***[III. Wireless Man-In-The-Middle Attacks](http://solstice.sh/workshops/advanced-wireless-attacks/iii-wireless-man-in-the-middle-attacks/)***
   * [IV. SMB Relays and LLMNR/NBT-NS Poisoning](http://solstice.sh/workshops/advanced-wireless-attacks/iv-smb-relays-and-llmnr-nbt-ns-poisoning/)
   * [V. Firewall And NAC Evasion Using Indirect Wireless Pivots](http://solstice.sh/workshops/advanced-wireless-attacks/v-firewall-and-nac-evasion-using-indirect-wireless-pivots/)

---

# Chapter Overview

In [Attacking and Gaining Entry to WPA2-EAP Wireless Networks](), we used an Evil Twin attack to steal EAP credentials. This was a relatively simple attack that was performed primarily on Layer 2 of the OSI stack, and worked very well for its intended purpose. However, if we want to do more interesting things with our rogue access point attacks, we’re going to have to start working at multiple levels of the OSI stack.

Suppose we wanted to use an Evil Twin to perform a man-in-the-middle attack similar to ARP Poisoning. In theory, this should be possible since in an Evil Twin attack the attacker is acting as a functional wireless access point. Furthermore, such an attack would not degrade the targeted network in the same way that traditional attacks such as ARP Poisoning do. Best of all, such an attack would be very stealthy, as it would not generate additional traffic on the targeted network.

To be able to execute such an attack, we will need to expand the capabilities of our rogue access point to make it behave more like a wireless router. This means running our own DHCP server to provide IP addresses to wireless clients, as well as a DNS server for name resolution. It also means that we’ll need to use an operating system that supports packet forwarding. Finally, we’ll need a simple yet flexible utility that redirects packets from one network interface to another.

![evil twin attack](http://solstice.sh/images/workshops/awae/iii/wirelessmitm.png)


# Configuring Linux As A Router

Before we begin, execute the following commands to prevent extraneous processes from interfering with the rogue access point.

	service network-manager stop
	rfkill unblock wlan
	ifconfig wlan0 up

For our access point, we’ll use hostapd once again. Technically, we don’t have much choice in the matter if we continue to use Linux as an attack platform. This is because hostapd is actually the userspace master mode interface provided by mac80211, which is the wireless stack used by modern Linux kernels.

{% highlight bash %}

	interface=wlan0
	driver=nl80211
	ssid=FREE_WIFI
	channel=1
	hw_mode=g

{% endhighlight %}

Hostapd is very simple to use and configure. The snippet included above represents a minimal configuration file used by hostapd. You can paste it into a file named hostapd.conf and easily create an access point using the following syntax.  

	hostapd ./hostapd.conf

After starting our access point, we can give it an IP address and subnet mask using the commands shown below. We’ll also update our routing table to allow our rogue AP to serve as the default gateway of its subnet.

	ifconfig wlan0 10.0.0.1 netmask 255.255.255.0
	route add -net 10.0.0.0 netmask 255.255.255.0 gw 10.0.0.1

For DHCP, we can use either dhcpd or dnsmasq. The second option can often be easier to work with, particularly since it can be used as a DNS server if necessary. A typical dnsmasq.conf file looks like this:

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

The first entry in the snippet shown above defines a DHCP pool of 10.0.0.80 through 10.0.0.254.  The second two entries are DHCP Options that are used to tell clients where to find the nameserver and network gateway. The dhcp-authoritative flag specifies that we are the only DHCP server on the network. The log-queries entry is self-explanatory.

Copy the config snippet shown above into a file named dnsmasq.conf, and run in a new terminal using the following syntax. By default, dnsmasq binds to the wildcard address. Since we don't want dnsmasq to do this, we keep it from doing so using the -z flag. Additionally, we use the -i flag to force dnsmasq to only listen on our $phy interface. We use the -I flag to explicity forbid dnsmasq from running on our local interface. The -p flag is used to indicate the port on which dnsmasq should bind when acting as a DNS server. Setting the -p flag to 0 instructs dnsmasq to not start its DNS server at all.

	dnsmasq -z -p 0 -C ./dnsmasq.conf -i "$phy" -I lo

We have an access point, a DNS server, and a DHCP server. To enable packet forwarding in Linux, we use the proc filesystem as shown below.

	echo ‘1’ > /proc/sys/net/ipv4/ip_forward

Finally, we configure iptables to allow our access point to act as a NAT. Iptables is a userspace utility that allows administrators to configure the tables of the Linux kernel firewall manually. This is by far the most interesting yet complicated part of this attack. We begin by setting the default policy for the INPUT, OUTPUT, and FORWARD chains in iptables to accept all packets by default. We then flush all tables to give iptables a clean slate.

	iptables --policy INPUT ACCEPT
	iptables --policy FORWARD ACCEPT
	iptables --policy OUTPUT ACCEPT
	iptables --flush
	iptables --table nat --flush

We then append a new rule to the POSTROUTING chain of the nat table. Any changes made to the packet by the POSTROUTING chain are not visible to the Linux machine itself since the chain is applied to every packet before it leaves the system. The rule chain that we append to POSTROUTING is called MASQUERADE. When applied to a packet, the MASQUERADE rule chain sets the source IP address to the outbound NIC’s external IP address. This effectively creates a NAT. Unlike the SNAT rule chain, which serves a similar function, the MASQUERADE rule chain determines the NIC’s external IP address dynamically. This makes it a great option when working with a dynamically allocated IP address. The rule also says that the packet should be sent to eth0 after the MASQUERADE rule chain is applied.

	iptables --table nat --append POSTROUTING -o $upstream -- jump MASQUERADE

To summarize, the command shown above tells iptables to modify the source address of each packet to eth0’s external IP address and to send each packet to eth0 after this modification occurs.

![evil twin attack](http://solstice.sh/images/workshops/awae/iii/image-2.png)

In the diagram shown above, any packets with a destination address that is different from our rogue AP’s local address will be sent to the FORWARD chain after the routing decision is made. We need to add a rule that states that any packets sent to the FORWARD chain from wlan0 should be sent to our upstream interface. The relevant command is shown below.

	 iptables --append FORWARD -i $phy -o $upstream --jump ACCEPT

That’s everything we need to use Linux as a functional wireless router. We can combine these commands and configurations into a single script that can be used to start a fully functional wireless hotspot. Such a script can be found in the ~/awae/lab2 directory in your Kali VM, as well as at the following URL.

 - [https://github.com/s0lst1c3/awae/blob/master/lab2/hotspot.sh](https://github.com/s0lst1c3/awae/blob/master/lab2/hotspot.sh)

# Lab Exercise: Using Linux As A Router

For this exercise, you will practice using your Kali VM as a functional wireless hotspot.

1. Begin by ensuring that your host operating system has a valid internet connection.
2. From your Kali VM, use the bash script in your ~/awae/lab2 directory to create a wireless
hotspot.
3. From either your cell phone or your Windows AD Victim VM, connect to your wireless
hotspot and browse the Internet. In your Kali VM, observe the hostapd and dnsmasq
output that appears in your terminal.

# Classic HTTP Downgrade Attack

Now that we know how to turn our Linux VM into a wireless router, let’s turn it into a wireless router that can steal creds. We’ll do this by using iptables to redirect all HTTP and HTTPS traffic to a tool called SSLStrip. This tool will perform two essential functions. First, it will create a log of all HTTP traffic sent to or from the victim. We can then search this log for credentials and other sensitive data. Second, it will attempt to break the encryption of any HTTPS traffic it encounters using a technique called SSL Stripping [6].

SSL Stripping was first documented by an excellent hacker known as Moxie Marlinspike. In an SSL Stripping attack, the attacker first sets up a man-in-the-middle between the victim and the HTTP server. The attacker then begins to proxy all HTTP(S) traffic between the victim and the server.  When the victim makes a request to access a secure resource, such as a login page, the attacker receives the request and forwards it to the server. From the server’s perspective, the request appears to have been made by the attacker [6].

Consequently, an encrypted tunnel is established between the attacker and the server (instead of between the victim and the server). The attacker then modifies the server’s response, converting it from HTTPS to HTTP, and forwards it to the victim. From the victim’s perspective, the server has just issued it an HTTP response [6].

![evil twin attack](http://solstice.sh/images/workshops/awae/iii/sslstrip.png)

All subsequent requests from the victim and the server will occur over an unencrypted HTTP connection with the attacker. The attacker will forward these requests over an encrypted connection with the HTTP server. Since the victim’s requests are sent to the attacker in plaintext, they can be viewed or modified by the attacker [6].

The server believes that it has established a legitimate SSL connection with the victim, and the victim believes that the attacker is a trusted web server. This means that no certificate errors will occur on the client or the server, rendering both affected parties completely unaware that the attack is taking place [6].

Let’s modify our bash script from [Configuring Linux As A Router]() so that it routes all HTTP(S) traffic to SSLStrip. We’ll do this by appending a new rule to iptables’ PREROUTING chain. Rules appended to the PREROUTING chain are applied to all packets before the kernel touches them.  By appending the REDIRECT rule shown below to PREROUTING, we ensure that all HTTP and HTTPS traffic is redirected to a proxy running on port 10000 in userspace [7][8].

	iptables --table nat --append PREROUTING --protocol tcp --destination-port 80 --jump REDIRECT --to-port 10000

We then add the following call to SSLStrip, using the -p flag to log only HTTP POST requests.

	python -l 10000 -p -w ./sslstrip.log 

The updated bash script can be found on your Kali VM in the ~/awae/lab3 directory, as well as at
the following URL:

 - [https://github.com/s0lst1c3/awae/blob/master/lab3/http-downgrade.sh](https://github.com/s0lst1c3/awae/blob/master/lab3/http-downgrade.sh)

# Lab Exercise: Wireless MITM With And HTTP Downgrade

Let’s use the script we wrote in the last section to perform a wireless Man-in-the-Middle attack
using an Evil Twin and SSLStrip.

1. Begin by ensuring that your host operating system has a valid internet connection.
2. Create an account at https://wechall.net using a throwaway email address.
3. If currently authenticated with https://wechall.net, logout.
4. Create an open network named “FREE\_WIFI” using your wireless router
5. From your Windows AD Victim VM, connect to “FREE\_WIFI” and browse the Internet.
6. From your Kali VM:
	1. Use the updated bash script to perform an Evil Twin attack against “FREE\_WIFI”
	2. Observe the output that appears in your terminal when the Windows AD Victim VM connects to your rogue access point
7. From your Windows AD Victim VM:
	1. Browse the internet, observing the output that appears in your terminal
	2. Navigate to [http://wechall.net](http://wechall.net).
	3. Authenticate with http://wechall.net using the login form to the right of the screen.
8. From your Kali VM:
	1. As your authentication attempt occurred over an unencrypted connection, your WeChall credentials should now be in ./sslstrip.log. Find them.
	2. From your Windows AD Victim VM:
	3. Logout of [http://wechall.net](http://wechall.net)
	4. Navigate to [https://wechall.net](https://wechall.net)
	5. Authenticate with [https://wechall.net](https://wechall.net) using the login form to the right of the screen.
9. Despite the fact that your authentication attempt occurred over HTTPS, your WeChall credentials should have been added to ./sslstrip.log a second time. Find this second set of credentials.

# Downgrading Modern HTTPS Implementations Using Partial HSTS Bypasses

Before beginning this section, repeat [Lab Exercise: Wireless MITM Using Evil Twin and SSLStrip]() using your Twitter account. You should notice that the attack fails. This is due to a modern SSL/TLS implementation known as **HSTS**.

HSTS is an enhancement of the HTTPS protocol that was designed to mitigate the weaknesses exploited by tools such as SSLStrip [9]. When an HTTP client requests a resource from an HSTS enabled web server, the server adds the following header to the response:

	Strict-Transport-Security: max-age=31536000

This header tells the browser that it should always request content from the domain over HTTPS.  Most modern browsers maintain a list of sites that should always be treated this way [10]. When the web browser receives HSTS headers from a server, it adds the server’s domain to this list. If the user attempts to access a site over HTTP, the browser first checks if the domain is in the list.  If it is, the browser will automatically perform a 307 Internal Redirect and requests the resource over HTTPS instead [9].

The IncludeSubdomains attribute can be added to HSTS headers to tell a web browser that all subdomains of the server’s domain should be added to the list as well [9]. For example, suppose a user attempts to access the following URL:

	https://evilcorp.com

If the server responds with the following HSTS headers, the user’s browser will assume that any request to \*.evilcorp.com should be loaded over HTTPS as well.

	Strict-Transport-Security: max-age=31536000; includeSubDomains

Additionally, site administrators have the option of adding their domain to an HSTS preload list that ships with every new version of Firefox and Google Chrome. Domains included in the HSTS preload list are treated as HSTS sites regardless of whether the browser has received HSTS headers for that domain.

HSTS is an effective means of protecting against SSL Stripping attacks. However, it is possible to perform a partial-HSTS bypass when the following conditions are met:

1. The server’s domain has not been added to the HSTS preload list with the IncludeSubdomains attribute set.
2. The server issues HSTS headers without the IncludeSubdomains attribute set.

The following technique was first documented by LeonardoNve during his BlackHat Asia 2014 presentation OFFENSIVE: Exploiting changes on DNS server configuration [11]. To begin, the attacker first establishes a man-in-the-middle as in the original SSL Stripping attack. However, instead of merely proxying HTTP, the attacker also proxies and modifies DNS traffic. When a victim navigates to www.evilcorp.com, for example, the attacker redirects the user to wwww.evilcorp.com over HTTP. Accomplishing this can be as simple as responding with a 302 redirect that includes the following location header:

	Location: http://wwww.evilcorp.com

The user’s browser then makes a DNS request for wwww.evilcorp.com. Since all DNS traffic is proxied through the attacker, the DNS request is intercepted by the attacker. The attacker then responds using his or her own DNS server, resolving wwww.evilcorp.com to the IP address of www.evilcorp.com. The browser then makes an HTTP request for wwww.evilcorp.com. This request is intercepted by the attacker and modified so that it is an HTTPS request for www.evilcorp.com. As in the original SSL Stripping attack, an encrypted tunnel is established between the attacker and www.evilcorp.com, and the victim makes all requests to wwww.evilcorp.com over plaintext [11].

![evil twin attack](http://solstice.sh/images/workshops/awae/iii/hsts-bypass.png)

This technique is effective provided that certificate pinning is not used, and that the user does not notice that they are interacting with a different subdomain than the one originally requested (i.e. wwww.evil.com vs www.evilcorp.com). To deal with the second issue, an attacker should choose a subdomain that is believable within the context in which it is used (i.e. mail.evilcorp.com should be replaced with something like mailbox.evilcorp.com).  Let’s update our bash script so that it performs a partial HSTS bypass using LeonardoNve’s DNS2Proxy and SSLStrip2. We do this by first adding a line that uses iptables to redirect all DNS traffic to dns2proxy.

	iptables --table nat --append PREROUTING --protocol udp - -destination-port 53 --jump REDIRECT --to-port 53

We then replace our call to SSLStrip with a call to SSLStrip2.

	python /opt/sslstrip2/sslstrip2.py -l 10000 -p -w ./sslstrip.log &

Finally, we add a call to dns2proxy as shown below.

	python /opt/dns2proxy/dns2proxy.py –i $phy &

Our completed bash script can be found in your Kali VM within the ~/awae/lab4 directory, as well as at the following URL:

 - [https://github.com/s0lst1c3/awae/blob/master/lab4/partial-hsts-bypass.sh](https://github.com/s0lst1c3/awae/blob/master/lab4/partial-hsts-bypass.sh)

# Lab Exercise: Wireless MITM With Partial HSTS Bypass
1. Populate your browser’s HSTS list by attempting to login to [Bing.com]()
2. Repeat Lab Exercise: Wireless MITM Using Evil Twin and SSLStrip using the completed bash script. Instead of capturing your own WeChall credentials, capture your own Bing credentials as you attempt to login to Bing.com instead. You should notice requests to wwww.bing.com as you do this.

---

### Next chapter: *[IV. SMB Relays and LLMNR/NBT-NS Poisoning](http://solstice.sh/workshops/advanced-wireless-attacks/iv-smb-relays-and-llmnr-nbt-ns-poisoning/)*

---
