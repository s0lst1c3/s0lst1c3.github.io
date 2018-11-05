---
layout: post
title: eaphammer Update: support for Password Spraying as of v0.4.0-beta
categories:
- wireless
- eaphammer
---

As a version 0.4.0, EAPHammer supports the ability to perform password spraying attacks against WPA2-EAP wireless networks. Password spraying is a type of bruteforce attack in which multiple successive login attempts are made using a single password and a large number of accounts. A recent study published in May 2018 by GCHQ's National Cyber Security Centre (NCSC) revealed that 75% of participating organizations had accounts with passwords featured in top-1000 password lists. With this in mind, checking for weak credentials and password reuse is a much needed capability during any wireless security assessment.

## Example Use Case
Suppose you are conducting a wireless security assessment and somehow stumble across a list of 1000 RADIUS account names. As of eaphammer v0.4.0, you can use the following command to check if any of these accounts are protected by the password "letmein" (ranked #7 in prevelence as of 2018):

	./eaphammer --interface-pool wlan0 wlan1 wlan2 wlan3 --essid example-corp --password letmein --user-list users.txt

![eap-spray](http://solstice.sh/images/eap-spray/split_panes_full.png)

The latest version of eaphammer can be found at: [https://github.com/s0lst1c3/eaphammer] (https://github.com/s0lst1c3/eaphammer)
