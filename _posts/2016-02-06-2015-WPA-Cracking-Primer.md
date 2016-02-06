---
layout: post
title: WPA Cracking Primer - Aircrack
categories:
- wireless
- scripting
---

![Free WiFi]({{ site.baseurl }}images/wpa-cracking/free-wif am i.jpg)


# Prerequisites


For this tutorial you will need a laptop running Linux, an external wireless adapter that supports packet injection, and a cheap wireless router to practice on. Any Linux distro will do, so long as it supports the drivers for your wireless cards. As far as wireless cards go, I personally enjoy using the [TP-Link TL-WNT722N](http://www.amazon.com/TP-LINK-TL-WN722N-Wireless-Adapter-External/dp/B002SZEOLG/ref=sr_1_1?s=pc&ie=UTF8&qid=1446340003&sr=1-1&keywords=tp-link+wn722n) because it is reliable and cheap. Ebay is a good place to find used networking equipment to practice on.

Once we have our physical prerequisites out of the way, we're going to need to install [aircrack-ng](http://www.aircrack-ng.org/), [reaver](https://code.google.com/p/reaver-wps/), and [GNU macchanger](https://github.com/alobbs/macchanger). We can do so by running the following commands in our terminal:

	# Fedora 22/23
	sudo dnf install macchanger aircrack-ng reaver

	# Fedora 21/20/19/18 and CentOS/RHEL 7.1
	sudo yum install macchanger aircrack-ng reaver

	# Ubuntu/Debian/Backtrack/Kali
	sudo apt-get install macchanger aircrack-ng reaver

# Killing Daemons

Before we get started, let's bring up a root shell by using the following command in our terminal:

	# logically equivalent to sudo $SHELL
	sudo -s

Next we need to disable and stop the NetworkManager service, because it will interfere with just about everything we are about to attempt:

	# ubuntu/debian/backtrack/kali
	service network-manager stop
	service network-manager disable
	service network-manager status

	# centos/arch/fedora/rhel
	systemctl stop NetworkManager
	systemctl disable NetworkManager
	systemctl status NetworkManager

We should also make sure that dhclient is not running:

	killall dhclient

	# for systems that do not have 'killall' installed
	for i in `pgrep dhclient`; do kill $i; done

# Naming our wireless NICs

Once we've killed NetworkManager and dhclient, we use __iwconfig__ to obtain the names of our wireless adapters. Your system's interface naming scheme will vary depending on your Linux distro. If you're using REHL, Centos, or Fedora the name of your wireless adapter will start with 'wlp' followed by a seemingly arbitrary string of characters. For example:

	wlp3s0
	wlp0s29u1u5
	wlp0s20u2

Most other distributions will have a naming convention that is much less insane: wireless cards start with the word 'wlan' and are immediately followed by a number. For example:

	wlan0
	wlan1
	wlan2
	.
	.
	.
	wlanN 

To get the name of your internal wireless card, first make sure your external wireless card is unplugged. Then run __iwconfig__ in your terminal to get a list of all network adapters on your system as shown below.

![listing nics 1]({{ site.baseurl }}images/wpa-cracking/list-nics1.png)

Check the output of __iwconfig__ for something that looks like one of the names outlined above. That's your internal wireless adapter. In the screenshot shown above, the internal wireless card is named __wlp3s0__.

We now have the name of our internal wireless card. To get the name of our external wireless card, we plug it back into our laptop and run __iwconfig__ again.

![listing nics 2]({{ site.baseurl }}images/wpa-cracking/list-nics2.png)

Look for the new wireless card that's been added to the list. That's the name of your external card. In the screenshot shown above, the external wireless card is named __wlp0s29u1u5__. 

# Attack Setup

First we disable our internal wireless card using __ifconfig__:

	# substitute wlp3s0 with the name of your internal NIC
	ifconfig wlp3s0 down

Next we give our external wireless card a random mac address and place it into monitor mode using __macchanger__, __ifconfig__, and __iwconfig__. Just substitute wlp0s29u1u5 with the name of your external wireless card.

	# disable external wireless interface 
	ifconfig wlp0s29u1u5 down
	
	# give external wireless interface a random mac address
	macchanger -r wlp0s29u1u5

	# place external wireless card into monitor mode
	iwconfig wlp0s29u1u5 mode monitor 

	# enable external wireless interface
	ifconfig wlp0s29u1u5 up

# Recon

With our external wireless card running in monitor mode, we can use airodump-ng to get a list of all nearby wireless access points and clients:

	airodump-ng wlp0s29u1u5
	
The output in your terminal will look something like this.

![airodump-ng]({{ site.baseurl }}images/wpa-cracking/airodump.png)

Nearby wireless access points are listed at the top. Each wireless access point is listed with 10 fields. The fields relevent to this tutorial are listed below:

- __BSSID__ - The mac address of the access point
- __PWR__ - The signal strength of the access point from our current location.
- __Beacons__ - The number of beacon frames we have received for the access point. 
- __\#Data__ - The number of data frames we have captured for the access point.
- __CH__  - The channel the access point is using.
- __ENC__ - The type of encryption used by the access point. In the example above, we see networks with no encryption (OPN), Wi-Fi Protected Access 2 (WPA2), and Wired Equivalent Privacy (WEP).
- __AUTH__ - The type of authentication used to secure the wireless access point. In our example, all protected access points are using Pre-Shared Keys (PSK) for authentication.
- __ESSID__ - The name of our wireless network. "Hidden" networks will show only the length of the ESSID, not the ESSID itself.

Nearby wireless client devices are enumerated at the bottom of airodump-ng's output. Each wireless client is listed with 7 fields. The relevent fields are:

- __BSSID__ - If the wireless client is currently associated with a wireless access, the mac address of the wireless access point is shown here. The value of this field corresponds to the BSSID of one of the nearby access points listed at the top of airodump-ng's output.
- __STATION__ - The mac address of the client's wireless network adapter.
- __PWR__ - The signal strength of the device from our current location.
- __Frames__ - The number of frames we have captured from the device.
- __Probe__ - The ESSID fields of any probe requests captured from the device are listed here. This allows us to know the device's preferred networks.

![airodump-ng]({{ site.baseurl }}images/wpa-cracking/airodump.png)

We need to find an access point used by our target network. For this tutorial, we are targetting a hidden network with ESSIDs - 'WARRIOR2' and 'WARRIOR5'. There are six access points with hidden ESSIDs in our vicinity. There are six characters in 'WARRIOR5' and 'WARRIOR2'. Looking at airodump-ng's output, we can see that there is only one hidden ESSID with a length of 8 characters. Therefore we can assume that the access point listed with BSSID 74:D0:2B:3F:06:88 belongs to the target network.

# Capturing the Handshake

Once we have identified an access point belonging to the target network, we proceed to capturing the 4-way WPA hanshake by devices to authenticate with the network.

![Dictionary Meme]({{ site.baseurl }}images/wpa-cracking/dictionary-meme.jpg)

Let's start out by killing our existing airodump-ng session, then starting a targetted airodump-ng session using the following command:

	airodump-ng -c 6 -w lol -–bssid 74:D0:2B:3F:06:88 wlp0s29u1u5

The -c flag tells airodump-ng to only capture packets associated with access points using channel 6. The -w lol flag allows us to write to a capture file named lol.cap. The --bssid flag tells airodump-ng to only capture packets associated with a specific BSSID. Finally, we specify the capturing interface as our last argument.

As soon as we run command shown above, we count to 60 then hit the space bar to pause airodump-ng's output.

![targeted airodump-ng]({{ site.baseurl }}images/wpa-cracking/airodump-targeted1.png)

Based on airodump-ng's output, we know that we were able to capture 57 beacons and 146 data packets in one minute. We can get a good estimate of network activity using the following formulas:

	beacon frames captured / 60 seconds == beacon frames per second
	data frames captured / 60 seconds == data frames per second

Our target network is running with approximately 2.43 data frames and .96 beacons per second. If the number of frames per second is low, it's a good indication that there aren't any devices currently active on the target network. There are two courses of action to take when this happens. You can sit tight and kill time until traffic increases, or simply leave and come back at a time when you think the network will be more active. Choose an option based on your attention span and whether you are prepared to stake out the target for hours to days on end without being noticed.

Once we have determined that the network is actually in use, we can proceed to capturing the 4-way WPA handshake. Let's resume our airodump-ng session by hitting the space bar.

In order to perform a WPA Dictionary attack, we need to capture the 4-way WPA handshake that is made when a device authenticates with the access point. We can do this by waiting for a device to successfully authenticate with our target access point, but this may take a long time. A faster option is to deauth a device that has already authenticated with the target access point, then capture the 4-way WPA handshake as it automatically tries to reconnect. Let's go with the second option and launch a deauth attack on the target network with the following command:

	aireplay-ng -0 5 -a 74:D0:2B:3F:06:88 wlp0s29u1u5

The -0 5 parameter tells aireplay-ng to send five deauth packets to the target BSSID. The -a 74:D0:2B:3F:06:88 parameter tells aireplay-ng to set the target BSSID to 74:D0:2B:3F:06:88. Finally, the name of our external wireless card is passsed as our last argument. 

![handshake]({{ site.baseurl }}images/wpa-cracking/deauth.png)

Typically, sending 5 deauth packets is sufficient. All associated clients will disconnect for a couple of seconds, then will attempt to reauthenticate with the WPA access point. When they do, airodump-ng will automatically capture the WPA 4-way handshake, as shown below:

![handshake]({{ site.baseurl }}images/wpa-cracking/handshake.png)

# Cracking the Capture File

Once we have the 4-way WPA handshake, we can close airodump-ng and proceed to crack our lol.cap file. The five deauth packets we sent in the last section are the only data we sent to our target. All of the other steps we have taken so far are completely passive. The actual dictionary attack, which we are about to perform, is performed entirely offline. As you can see, this is an incredibly stealthy attack.

Let's use following command to run a bruteforce attack against the capture file we made using airodump-ng:

	aircrack-ng -w rockyou.txt -b 74:D0:2B:3F:06:88 *.cap

The -w flag specifies the worldist that we want to use. I'm using the infamous rockyou wordlist, which can be found at [skullsecurity.org](https://wiki.skullsecurity.org/index.php?title=Passwords). The -b flag specifies the mac address of the target access point. Finally, we pass the name of our capture file as the last argument to aircrack-ng.

![cracking wpa]({{ site.baseurl }}/images/wpa-cracking/cracking-wpa.png)
note - cracking-wpa.png seems to have turned into a blank png file. weird. have to retake screenshot

Depending on the strength of the target's password, this step could take anywhere from seconds to days to complete. It is unfeasable to crack strong WPA passwords without extraordinary computing resources. This limits us to hacking WPA networks protected by weak to moderately strong passphrases, which is approximately 40% of them. 

# Accessing the Network

Now that we have recovered the WPA preshared key, let's go over how to quickly connect to the target network from tthe command line. 

We start out by re-enabling our internal wireless card and giving it a fake mac address. Remember to substitute wlp3s0 for the name the name of your internal NIC.

	# enable internal wireless NIC
	macchanger -r wlp3s0
	ifconfig wlp3s0 up

We then use __wpa_passphrase__ to create a config file using the creds we just stole. In the below example, substitute "WARRIOR2" and "6d-51%_OP9Jzv!" for name and passphrase used by your target network:

	wpa_passphrase WARRIOR2 6d-51%_OP9Jzv! > lol.conf 

Finally, we connect to our target network using __wpa_supplicant__. Once again, substitute wlp3s0 for your own wireless NIC as in the example below:

	wpa_supplicant -i wlp3s0 -c lol.conf 

We should now be connected to the target network. To obtain an ip address, use __dhclient__:

	# substitute wlp3s0 for the name of your internal NIC
	dhclient wlp3s0

That's all there is to it. At this point we can proceed to running man in the middle attacks, abusing the router itself, or enjoying our free wifi. Here's a video demonstrating what the attack should look like:

<iframe width="560" height="315" src="https://www.youtube.com/embed/y2Jn12rOeP0" frameborder="0" allowfullscreen></iframe>


# Bibliography

https://sviehb.files.wordpress.com/2011/12/viehboeck_wps.pdf
http://aspj.aircrack-ng.org/
