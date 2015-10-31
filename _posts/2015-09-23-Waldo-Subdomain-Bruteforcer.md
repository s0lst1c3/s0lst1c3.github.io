---
layout: post
title: Webapp Enumeration with Waldo
categories:
- tools
- web hacking
- enumeration
- red team labs
---

![waldo usage]({{ site.baseurl }}images/waldo-usage.png)

At Red Team Labs, we find ourselves using DirBuster a lot. It's a pretty
essential tool for quickly enumerating subdomains and web directories. The project
is no longer actively maintained however, and since it's written in Java it doesn't
exactly work well with our existing toolset. To deal with this, we wrote our own
multithreaded subdomain and directory bruteforcer in Python. We named it Waldo. You
can check out the project on github by clicking [here](http://www.github.com/red-team-labs/waldo).


![Demo]({{ site.baseurl }}images/waldo-demo.png)

_waldo -m s -d starbucks.com -t 2_

Waldo reads a list of possible subdomains or directories from a worldist file, and checks
the target domain to see if they are valid. It then prints the server response code and
associated ip address for each valid result. Waldo also produces a log file for each
session containing a grepable version of the data displayed in the terminal.

Waldo comes prebaked with its own default wordlist, but you can also specify your own
custom list should you feel so inclined.  By default, Waldo runs with 5 worker threads
and 1 file I/O thread. This should be fast enough for most use cases, but you can
manually set the number of worker threads with the -t flag. We got pretty fast results
in the above example with just two worker threads.

The screenshot above shows Waldo running in subdomain mode. As you can see, the tool
was able to find some pretty interesting results. Let's take a look at "e.starbucks.com."

![example 1]({{ site.baseurl }}images/e-starbucks.png)

Well that's something you don't see every day. Maybe a promo code or something? Who knows.
Let's checkout "intl.starbucks.com."

![example 2]({{ site.baseurl }}images/intl-starbucks.png)

Nice. Wonder how long that's been there. 


For more cool stuff from __Red|Team|Labs__ checkout [red-team-labs.com](https://red-team-labs.com), or go to [github.com/red-team-labs](http://github.com/red-team-labs).
