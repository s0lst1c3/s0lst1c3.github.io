---
layout: post
title: CSAW Quals 2015 - Lawn Care Simulator Writeup
categories:
- web hacking
- csaw quals 2015
- writeups
---

When we navigate to the challenge at http://54.165.252.74:8089 in our browser, we're
greeted with this fantastic looking page:

![Index page]({{ site.baseurl }}images/index.png)

It's a game, where presumably the objective is to watch grass grow. Awesome. Let's try
to log in.

The first thing we want to do is click the link at the bottom of the page that says
"Sign up for Lawn Care Simulator Premium." Doing so will take us to the following
registration page:

![Index page]({{ site.baseurl }}images/register.png)

If you try to sign up, you'll get the the following message:

![register failed]({{ site.baseurl }}images/register-failed.png)

Clearly we can't just register for the site, so let's poke around a little more and see if we can find another way in. Let's first go back to the index page and take a look at the source code:

![index source]({{ site.baseurl }}images/index-source.png)

Line 16 is particularly interesting. It contains an $.ajax call to '.git/refs/head/master', which means that there is a .git directory that we can possibly use to steal the source code for the web app. Let's see if we can at least get read access by attempting to access some common files found in a git repo:

![git config]({{ site.baseurl }}images/git-config.png)

Sweet. We have read access to the .git/config file. Let's try accessing .git/logs/HEAD:

![git log]({{ site.baseurl }}images/git-head.png)

It seems we also have read access to the log files, and that the .git repo does contain the web app's source code. We have already been able to access at least two of the files in .git, so it is likely that we have read access to every file in the repository. If this is true, it means that we can reconstruct the entire git repo on our local machine and analyze the app's source to look for vulnerabilities.

While it is possible to do this by hand, automating the process would be a much better use of our time. We can do this using the DVCS-Pillage toolset, which contains a script that can do this in seconds. First, clone the DVCS-Pillage git repo, which can be found here: [github.com/evilpacket/DVCS-Pillage](https://github.com/evilpacket/DVCS-Pillage)


![install git pillage]({{ site.baseurl }}images/install-git-pillage.png)

Now let's use gitpillage to steal the app's source code:

![run git pillage]({{ site.baseurl }}images/run-git-pillage1.png)

We then cd into our new new 54.165.252.74:8089 directory and look around:

![run git pillage]({{ site.baseurl }}images/run-git-pillage2.png)

We have our source code. Let's open up premium.php:

![premium.php]({{ site.baseurl }}images/premium-source.png)

It looks like we need to gain access to 54.165.252.74:8089/premium.php in order to retrieve the flag. For that, we're going to need a valid username and a valid password.  

To authenicate, premium.php first checks to see if both 'password' and 'username' exist as POST parameters (line 15). If they do, then the script makes a call to validate(), passing the username and password as arguments. Since validate() is defined within validate_pass.php, let's take a look at that:

![validate_pass.php]({{ site.baseurl }}images/validate-pass-source.png)

Let's figure out what this source code is doing. The script first establishes a
connection to the database:

![loading the database]({{ site.baseurl }}images/load-database.png)

It then makes a call to mysql\_real\_escape\_string() to filter potentially dangerous
characters from $user. The variable $pass is not sanitized because it is never actually
used as part of the mysql query.

![real_escape_string() ]({{ site.baseurl }}images/real-escape-string.png)

The script then constructs a query from $user, and uses it to retrieve the user's hashed password from the database. The length of the hashed password, stored in the variable $hash, is then compared with the length of $pass. 

![query ]({{ site.baseurl }}images/query.png)

If the lengths of $pass and $hash are not the same, validate() returns False.

Finally, validate() uses a while loop to compare $hash with $pass, character by character. If $pass is equal to $hash, then the function returns True.

What we learn from this is that the app is hashing the password in the web browser, then sending it to the server where it is compared against the user's stored hash from the database.

Some key things we should take into consideration:

- both if statements use loose type comparisons (more information about that [here](http://www.owasp.org/images/6/6b/PHPMagicTricks-TypeJuggling.pdf))

- strlen() returns 0 if passed an empty string, and returns NULL if passed NULL as a paramter

- the statement 0 != NULL is false due to PHPs built in type juggling
	
- the while loop on line 16 will never execute if $hash[$index] is NULL

What happens if we pass empty strings as both the username and password? The mysql query on line 8 will fail, setting the value of $result to NULL. This will cause mysql\_fetch\_row() to return NULL on line 9, setting the value of $line to NULL. Because NULL is not an associative array, $hash will be set to NULL on line 10.  This means that strlen($hash) on line 12 will return NULL, and strlen($pass) will return 0. Since 0 != NULL evaluates to false, line 13 will never execute. The while loop on lines 16 through 22 will be bypassed as well, because $hash[$index] will evaluate to NULL, which is logically equivalent to false. Therefore the function will return true.

All we need to do is pass empty strings as both the username and the password. Sounds
easy enough. 

![attempting to enter blank password]({{ site.baseurl }}images/blank-pass.png)

We can't simply input empty strings through the front end, but we can send them
as POST parameters using a very small Python script:

	#!/usr/bin/python

	import requests
	
	URL = 'http://54.165.252.74:8089/premium.php'
	response = requests.post(URL, data={ 'password' : '', 'username' :' '})
	
	print response.text


Let's run the script and see what happens:

![got the flag!]({{ site.baseurl }}images/flag.png)

We have the flag! Very punny.
