---
title: "OS X Notifications from the Shell"
description: "Keeping myself from being too lazy."
layout: post
tags:
 - programming
 - tools
 - notifications
 - shell
---

I have one big productivity suck when working: asynchronous tasks. 

Just like in code, I think it's in pretty poor taste (not to mention inefficient) to start something and then just keep checking on it until you see it's done. Not only that, but it's easy to forget to check. 

This is what happens to me when dealing with my unit and integration tests. Maybe I'll go read an article while they're running or even drop into Reddit (which is a mistake in all cases), or if I'm a good boy I'll go update some documentation and style somewhere around my codebase that needs it.

In code, we've solved this problem with callbacks. Make an HTTP call and just tell it to let you know when the response comes back, instead of blocking everything else or continuously checking some value somewhere until it's changed to _complete_. It kind of dawned on me that I have a $3,000 callback machine in front of me; why not make it work for me? I get a notification when someone posts on my Facebook wall, when I'm tweeted at, and when my Amazon purchase ships -- why not get one when my tests are complete?

I'm not a big IDE fan, and I'm sure IDEs do this (I mean, Xcode does, of course) but when working in Python I like to work primarily in the command line when it comes to testing and execution. I did some research, and found that Apple exposes some Notification Center functionality to AppleScript, and I know that I can execute arbitrary AppleScript via the `osascript` command, so, _voilà_, I can post notifications from the shell.

All that was left is putting in some logic to display different messages based on parameters and whether or not the tests succeeded. Check out the gist below for a basic version of what I ended up implementing (my real version has some proprietary stuff in it, so I had to write a simpler version for demo purposes!).

<script src="https://gist.github.com/josefdlange/32be51f1efa9ec28bbba.js"></script>

I'd love some feedback, so please let me know what y'all think!