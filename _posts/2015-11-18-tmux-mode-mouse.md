---
title: "tmux 2.1 and mode-mouse"
description: "Poorly-documented breaking changes"
layout: post
categories: [programming, tools, documentation, changes]
---

I've been having an issue the last couple days that I couldn't figure out. I had upgraded most tools on my machine as I tend to do every once in a while, and after a few days I noticed that my mouse integration in `tmux` wasn't working quite the way I expected. Turns out, they had made a significant change to the mousing mode without documenting it very well, or at least without anything past putting in the changelog. I guess I should have read it, but the software should probably tell the user that a feature they're trying to use has been replaced.

At any rate, here's the config I had before, more or less:

{% gist f0042e428d6cc4571a20 %}

After the update to tmux 2.1, I found that I needed only this:

{% gist 85dfbf7383e276568fb5 %}

It's actually a lot more convenient, but it was frustrating that I had to search so far to find the solution. I'm by no means a tmux expert so if there's anything about this or other parts of my config that seem wrong or distasteful, I'd love to hear about it.
