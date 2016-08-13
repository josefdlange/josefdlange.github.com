---
layout: post
title: "Swizzling in Python"
categories: [programming, python]
---

### Story Time
Ever come across a Python library with some kind of shortcoming that you _know_ you can fix, but don't want to fork, fix, pull-request, wait five weeks, then update in order to get moving? Are you even too lazy to do just the first two steps and point your dependency manager at your own fork? Don't worry, you can probably fix it in code.

Right now I'm writing a Slack bot library that will support, in addition to the commonplace single command/response paradigm, a more conversational paradigm for multi-step commands and workflows. So, like a good programmer, I set up my module structure, my `setup.py`, and got moving. This was after a little bit of proof-of-concept code sketching, in which I came up with the general flow of connecting with the official Slack client for python, aptly named [`python-slackclient`](https://github.com/slackhq/python-slackclient). So, I ported the flow into my library's code, with some nicer structure, fired up a `virtualenv`, ran `python setup.py develop`, and then wrote a little wrapper script to invoke the client.

Boom, error. Weird, right? My proof-of-concept had worked fine...

After looking at Slack's code, I found that it was cleansing an exception from the [`websocket-client`](https://github.com/liris/websocket-client) library, where it was having problems locating my system's Root Certificate Authority file. That's a problem I don't really want to deal with right now. I have a feeling it has to do with the path magic that `python setup.py develop` does, so I'm willing to let it go. Nonetheless, I'm still stuck with it not working, and no way to tell the Slack client to get its head out of its ass and find the Root CA file for the websocket client.

Or is there?

### Investigation
The `websocket-client` method, `create_connection` takes a URL by default but also takes some `kwargs`, one of which is `sslopt`, a dictionary of values describing SSL options, _including_ the location of the Root CA file. Additionally, the `ssl` standard library includes methods for retrieving the location of said file. But, again, the calling of `create_connection` is within the Slack client's code! What do I do?

### _Swizzle that shit!_
Swizzling, for the uninitiated, is the same thing as monkey-patching, which is to say, replacing at runtime the implementation of a method on a class. Alas, it's a little more complex in this case, since the method I want to swizzle is on the class of an object, an instance of which is held by the Slack client. So, I ended up using a subclass of the client and overloading the initializer to dig into the client's state and swizzle the method in question.

Check it out!

{% gist 216ae1c59301fab5a54c1251a73c2396 %}

Note that you have to bind the newly-defined method to the _class_, meaning that if you only have a reference to an instance, you need to use the `__class__` attribute to get the class itself. Don't forget `self` as the first argument; this is important for the method to bind correctly to the class/instance.

Oh, so, by the way, that Slack bot library I'm working on can be found here: [SlackBorg](https://github.com/josefdlange/slackborg).