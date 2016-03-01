---
layout: post
title: Touchpad Protocol Reverse Engineering
date: '2011-11-17T22:11:00.001-06:00'
author: Seth Forshee
tags:
- Linux
- qemu
- VirtualBox
- touchpad
modified_time: '2011-11-18T01:10:34.238-06:00'
blogger_id: tag:blogger.com,1999:blog-8501269611012488187.post-4945479338213177026
blogger_orig_url: http://blog.forshee.me/2011/11/touchpad-protocol-reverse-engineering.html
---

Recently I was working on [adding Linux support](https://lkml.org/lkml/2011/11/7/433) for some undocumented touchpad protocols, and in the process I developed some useful reverse engineering techniques for touchpads. I'm documenting them here in hopes that they may be useful to others, or at least to myself at some point in the future.

First some background information about touchpads. After a reset, most touchpads use the old, reliable [PS/2 mouse protocol](http://www.computer-engineering.org/ps2mouse/). This allows the touchpad to function at a basic level without any special drivers, but most of the features that laptop owners have come to expect from touchpads are missing. Enabling those feature requires sending the touchpad a vendor-specific sequence of commands that switches the format of the data reported by the touchpad from the PS/2 mouse protocol to a proprietary data format that generally encodes much more information.

So to get a touchpad working in Linux as more than just a mouse we need two pieces of information: the magic command sequence to switch the touchpad to the proprietary data format and the layout of the proprietary data packets. Figuring out how to decode the data packets is tedious and time consuming, but it really isn't all that difficult. Mostly it involves doing different actions on the touchpad and seeing which bits change as a result, repeating until you've worked out the meaning of all (or at least most) of the data. The reverse engineering of the magic command sequence, however, is trickier.

In the absence of a specification, probably the only reasonable way to work out the initialization sequence for the touchpad is to observe how the Windows driver does it. The technique I settled on for doing this was to modify a virtual machine to replace the usual PS/2 mouse emulation with support for passing the raw PS/2 data between the guest OS and the touchpad, logging each byte of data as it passes through. Linux provides a driver named serio_raw that makes this pretty easy by providing a character device node that can be used to interact directly with the PS/2 port from userland.

At the end of this post you will find links to patches that enable this functionality in both qemu and VirtualBox. These are little more than quick-and-dirty hacks to get the functionality I needed, and the VirtualBox version is quite a bit more complex than the qemu version (and even still includes some debugging code). In order to use the patches you must first put the touchpad in serio mode (I have also included a script below that searches for a PS/2 mouse and changes the first one it finds to serio mode). When in serio mode the touchpad will be unusable to Linux, so having an external mouse is a must.

The patched virtual machine will look for two environment variables to enable passthrough to the touchpad and control data logging. PSMOUSE_SERIO_DEV_PATH must be set to the path of the serio_raw device node, usually /dev/serio_raw0. If it is not set or the device node cannot be opened the VM will fall back to the standard PS/2 mouse emulation. PSMOUSE_SERIO_LOG_PATH is used to specify the path to file used to log the protocol data. Without this variable the passthrough will still work, but no data will be logged.

The log file is structured as plain text with one line for each byte of data passed between the touchpad and the guest OS. Data is in hexadecimal, and bytes sent to the touchpad are prefixed with S while lines received from the touchpad are prefixed with R.

The method I would recommend is to first install the guest OS without the PSMOUSE_SERIO_DEV_PATH set. After successful installation, restart the VM with the environment variables set and verify that the guest can interact with the touchpad as a standard PS/2 mouse. Finally, install the driver and verify that the touchpad is fully functional. If it is, shut down the VM and have fun diving into the log!

I also identified one potential alternate technique for capturing touchpad protocol data under Windows that could be done in a bare-metal installation. If this technique worked it would likely be more convenient, as installing Windows under qemu and VirtualBox doesn't always go so smoothly (which is the reason I have patches for both virtual machines!).

Windows supports something called a [filter driver](http://en.wikipedia.org/wiki/Filter_driver) which, as I understand it, can sit transparently above or below another driver, filtering the data going between the driver and the adjacent layer in the stack. It may be possible to write such a driver that would be inserted below the vendor-supplied touchpad driver, logging all the PS/2 protocol data passing between the driver and the hardware.

I'm not at all familiar with the Windows driver architecture and haven't researched this in great detail to know whether or not it's actually possible. I thought I'd include it here in case someone knows of an existing tool that can do this or is up to the task of writing such a tool. If so, please drop me a line; I'd love to hear about it!

As promised, here are the files to make it all happen.

- [qemu-psmouse-serio-passthrough.patch](http://people.canonical.com/~sforshee/touchpad/qemu-psmouse-serio-passthrough.patch) : Patch to enable PS/2 mouse passthrough in qemu. Based on qemu-kvm_0.14.1+noroms-0ubuntu6 from Ubuntu Oneiric.
- [vbox-psmouse-serio-passthrough.patch](http://people.canonical.com/~sforshee/touchpad/vbox-psmouse-serio-passthrough.patch): Patch to enable PS/2 mouse passthrough in VirtualBox. Based on virtualbox_4.1.2-dfsg-1ubuntu1 from Ubuntu Oneiric.
- [mouse-to-serio.sh](http://people.canonical.com/~sforshee/touchpad/mouse-to-serio.sh): Script to change your touchpad to/from serio_raw mode. Run 'mouse-to-serio.sh 1' to place the mouse in serio_raw mode and 'mouse-to-serio.sh 0' to place it back into psmouse mode. Must be run as root.
