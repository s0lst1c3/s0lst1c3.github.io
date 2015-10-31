---
layout: post
title: OSX Tiling Workflow
categories:
- osx
- workflow
- javascript
---

I love tiling window managers. My absolute favorite is Awesome, which is available for both Linux and BSD. When I started spending more time in the OSX environment about a month ago, it quickly became apparent that keyboard centered workflows were a scarcity. Even worse, true tiling solutions for OSX were virtually nonexistent.

I decided to try out Slate after a bit of searching. Slate is an easily configurable 3rd party window manager. Just like Awesome, it’s completely programmable. Unlike Awesome however, its configs are not written in Lua. Instead, config files can either be written using Slate’s default syntax or in Javascript.

Slate’s Javascript API is extremely powerful, to the point where I was actually able to use it to closely mimic Awesome’s default behavior. To make this work, I used V Verma’s slate.js file as a starting point, as well a couple of 3rd party extras. Here’s how I did it.


##Step 1 - Replace Terminal with iTerm##

![split-panes full]({{ site.baseurl }}images/split_panes_full.png)

Although Apple’s default terminal is passable, it still leaves a lot to be desired. Thankfully, we can install iTerm2 to gain access to awesome features such as tmux style pane splitting, mouseless copy, and enhanced autocomplete. Just download and run the .dmg file from [iterm2.com/downloads.html](https://iterm2.com/downloads.html) and you’re good to go.  

##Step 2 - Make iTerm2 and Spaces more responsive, other tweaks##

Steve Jobs certainly had an eye for aesthetics, and OSX is full of animated effects that are simply beautiful. If you’re boring and utilitarian like me, this probably annoys the shit out of you. Although the graphical overhead in OSX does not seem to be a problem in most cases, the default animation settings make switching Spaces and typing in the terminal painfully slow. Let’s remedy that situation.

To set a freakishly fast keyboard repeat rate, just copy this into your terminal and run it:

	defaults write NSGlobalDomain KeyRepeat -int 0

To show IP address, hostname, and system information when clicking the clock in the login window, we can run this:

	sudo defaults write /Library/Preferences/com.apple.loginwindow AdminHostInfo HostName

To show hidden files by default:

	defaults write com.apple.finder AppleShowAllFiles -bool true; killall Finder

Turn off the prompt that is displayed when we exit iTerm2:

	defaults write com.googlecode.iterm2 PromptOnQuit -bool false; killall iTerm

The same thing goes for spaces. We can speed up the animation by running the following command:

	defaults write com.apple.dock expose-animation-duration -int 0; killall Dock

##Step 3 - Fake a Tiling Window Manager with Slate##

This is the cool part. Let’s start out by downloading and installing Slate by running the following command:

	cd /Applications && curl http://www.ninjamonkeysoftware.com/slate/versions/slate-latest.tar.gz | tar -xz

Slate should now be good to go. Now we just need to configure it to behave like Awesome:

	git clone git@github.com:s0lst1ce/tiling-slate.git && cp ~/tiling-slate/slate.js ~/.slate.js && rm -rf tiling-slate

Finally, just relaunch Slate as shown in the following screenshot:

![Source: blow.niw.at]({{ site.baseurl }}images/tumblr_inline.png)

o switch window layouts, just use the following key combination:

	control+command+space

You can also switch focus between windows by using these key combos:

	// switch focus up
	control+command+k
	
	// switch focus down
	control+command+j
	
	// switch focus right
	control+command+l
	
	// switch focus left
	control+command+h

These keystrokes should seem familiar if you’re a vim user.

##Step 4 - Install Afred##

The last thing we’re going to do is install Alfred, an efficient and keyboard centered application launcher. Just download it from [alfredapp.com](https://alfredapp.com/#download), extract everything from the zip file, and run the installer. Then check out the tutorial at [support.alfredapp.com/tutorials:your-first-5-minutes](http://support.alfredapp.com/tutorials:your-first-5-minutes) to set it up. 

##Credits##

This would not have been possible without the previous work of V Verma. He’s an awesome dude and a genius coder. Make sure to check out his website:

[www.vverma.net](http://www.vverma.net/)







