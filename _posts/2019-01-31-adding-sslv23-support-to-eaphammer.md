---
layout: post
title: EAPHammer Release 0.5.0v - Legacy Crypto Support
categories:
- wireless
- eaphammer
---

A couple of days ago, EAPHammer version 0.5.0 was released. This update introduces a very subtle, yet very important capability to the project: SSLv2/3 support.

[https://github.com/s0lst1c3/eaphammer/releases/tag/v0.5.0-beta](https://github.com/s0lst1c3/eaphammer/releases/tag/v0.5.0-beta)). 

Why is this update so important? To answer this question, let's first go over some background info at a very high level.

EAPHammer is a tool that creates rogue access points. In order to use it to assess the security of WPA/2-EAP networks, it needs to be able to create rogue access points that support EAP methods that rely on TLS and SSL, such as EAP-PEAP. To provide this support, EAPHammer relies on a library called OpenSSL.

OpenSSL itself is a highly configurable library (except when it isn't, as we'll soon learn) that implements many different versions of SSL and TLS. For each of these protocol versions, the library provides support for a variety of individual cipher suites and additional capabilities.

The problem is that many of these configuration options are insecure, particularly the ones that allow communication using older cipher suites and protocol versions. SSL versions 2 and 3 are particularly problematic, along with TLS version 1.0. In particular, the problems that affect SSL versions 2 and 3 are serious enough that support for these protocols has been explicitly stripped by default from the latest builds of OpenSSL. As a result, modern operating systems such as Windows 10 and the latest versions of Kali will refuse to communicate with legacy systems such as Windows 7 that still rely on SSLv2/3.

# How does this relate to EAPHammer?

Without SSLv3 support, tools such as EAPHammer have cannot communicate wirelessly with legacy systems such as Windows 7. These legacy systems will outright refuse to connect to wireless access points that do not support SSLv3, as shown in the following screenshot:

![Figure 1](http://s0lst1c3.github.io/images/eaphammer-sslv23/sslv3-windows7-old.png)

If you're going to be in the business of running wireless pentests against enterprise organizations, you're going to need to need tools that can execute rogue access point attacks against Windows 7 hosts. The operating system is still supported by Microsoft until 14 January 2020, and its presence within enterprise environments is unlikely to disappear anytime soon after that. Much of the BYOD wireless attack surface still relies on SSLv3 as well.


**Why isn't this an end-user problem?**

As an end-user, adding SSLv2/3 support to your offensive wireless toolkit is easier said than done. Your options are to:

- use a tool that relies on an older version of OpenSSL (such as v1.0.0). You'll get SSLv2/3 support, at the cost of losing your ability to attack modern platforms.
- replace your system-wide OpenSSL installation with a hand-compiled version that has been configured to offer SSLv2/3 support prior to the compilation process. Not only is this a PITA,  it's actually detrimental to the overall security posture of your operating system. *Which you should care about. Particularly if you handle client data.*

However, even these steps that can be taken by an end-user don't completely solve the problem. The latest version of hostapd (from which EAPHammer/Mana/WPE/Sniffair/etc are all based) actually includes code that explicitly disables SSLv2/3 at runtime.

**Why should I believe you?**

Looking at the source code for hostapd 2.6, we can use grep to perform a recursive and case-insensitive search for the words 'sslv2' and 'sslv3'.

![Figure 2](http://s0lst1c3.github.io/images/eaphammer-sslv23/grep-for-sslv3-and-sslv2.png)

This reveals the following lines of code within hostapd/src/crypto/tls\_openssl.c:

	./src/crypto/tls_openssl.c:973:	SSL_CTX_set_options(ssl, SSL_OP_NO_SSLv3);
	./src/crypto/tls_openssl.c:1352:	options = SSL_OP_NO_SSLv2 | SSL_OP_NO_SSLv3 |
	root@kali:~/eaphammer/hostapd-eaphammer# grep -rni SSLv2 .
	./src/crypto/tls_openssl.c:954:		ssl = SSL_CTX_new(SSLv23_method());
	./src/crypto/tls_openssl.c:972:	SSL_CTX_set_options(ssl, SSL_OP_NO_SSLv2);
	./src/crypto/tls_openssl.c:1352:	options = SSL_OP_NO_SSLv2 | SSL_OP_NO_SSLv3 |

From the file header located at the top of hostapd/src/crypto/cryto.h we know that hostapd/src/crypto/tls\_openssl.c contains wrapper code for functions defined within within libssl:

![Figure 3](http://s0lst1c3.github.io/images/eaphammer-sslv23/crypto-dot-h-header.png)

The SSL\_CTX\_set\_options() function shown in *Figure 2* has been included from libssl, and is used to configure libssl at runtime using bitmasks (see: [ https://www.openssl.org/docs/man1.0.2/ssl/SSL_CTX_set_options.html](https://www.openssl.org/docs/man1.0.2/ssl/SSL_CTX_set_options.html)). The global variables containing the bitmasks are defined by preprocessor directives in openssl/include/openssl/ssl.h:

![Figure 4](http://s0lst1c3.github.io/images/eaphammer-sslv23/libssl-no-sslv-def.png)

 In the snippets of code shown in *Figure 2*, the SSL\_CTX\_set\_options() function is being used to forbid libssl from supporting connections made using SSLv2/3.

In conclusion: merely using a build of OpenSSL that has been compiled with SSLv2/3 support isn't enough, because modern versions of hostapd explicitly forbid the use of these protocol versions at runtime.

**How has this issue been fixed within EAPHammer?**

EAPHammer now relies on its own local build of OpenSSL that exists independently of the build used by the operating system. This local OpenSSL build is linked to EAPHammer during the initial setup process, and is compiled with support for SSLv2/3 along with an array of weaker cipher suites that may be needed to communicate with legacy clients.This design decision does mean that EAPHammer's initial setup process takes significantly longer, but unless you have to perform "initial" setups regularly for some reason, the ends justify the means.

Additionally, EAPHammer's version of hostapd has been patched to allow SSLv2/3 support. Doing so was simply a matter of commenting out the offending lines of code shown in *Figure 2ch* previously. These changes to hostapd will also be submitted to related projects such as Mana and Hostapd-WPE.
