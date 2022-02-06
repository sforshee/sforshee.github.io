---
layout: single
title: 'Fixing 256-color Support with tmux in terminator'
tags:
- Linux
redirect_from: "/2016/05/26/fixing-256-color-support-in-tmux-with-terminator.html"
---

I use [terminator](http://gnometerminator.blogspot.com/p/introduction.html) for 
my terminal emulator. Generally everything works well, but one problem has been 
bothering me recently. My vim color scheme shows up fine in terminator but not 
from tmux within terminator. Everything is okay when using tmux in 
gnome-terminal though, so it seems to be the combination of tmux and terminator 
that is problematic.

After a little digging I found out that this is due to terminator setting 
`TERM=xterm` in the environment, causing tmux to think that the terminal doesn't 
support 256 colors. This is easily verified by the fact that running 
`TERM=xterm-256color tmux` fixes the issue.

If terminator supports 256 colors, why is it setting `TERM` to `xterm` instead 
of `xterm-256color`? I didn't really dig into it, but the gory details are 
[here](https://bugzilla.gnome.org/show_bug.cgi?id=640940) for anyone who is 
interested. The tl;dr is that it doesn't look like it will get fixed soon.

But since the terminal really does support 256 colors, a simple workaround is to 
just override `TERM` in the environment. This should only be done for terminator 
though, we don't want to mess with the variable as set by other terminal 
emulators.

This can be done in the terminator configuration. It's part of the profile 
settings though, so if you have more than one profile it needs to be done for 
each one.

In the gui you can open the preferences and navigate to `Profiles -> <profile> 
-> Command`. Check the box to run a custom command then enter 
`TERM=xterm-256color bash -l` as the custom command (assuming you use bash, if 
not substitute your shell). Or open `~/.config/terminator/config` and add this 
line to each profile:

    custom_command = TERM=xterm-256color bash -l

Newly launched terminator windows will have `TERM=xterm-256color`.
