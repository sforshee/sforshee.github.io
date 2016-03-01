---
layout: post
title: Container Mounts in Ubuntu 16.04
date: '2016-02-22T15:26:00.000-06:00'
author: Seth Forshee
tags: 
modified_time: '2016-02-22T15:50:10.907-06:00'
blogger_id: tag:blogger.com,1999:blog-8501269611012488187.post-2865962684540086615
blogger_orig_url: http://blog.forshee.me/2016/02/container-mounts-in-ubuntu-1604.html
---

Something I've been working on for a while now is mounting select filesystems from within unprivileged (i.e. user namespace) containers. Though I'm still working on getting this into upstream Linux, support for mounting fuse and ext4 was recently merged as an opt-in feature in the xenial kernel. The instructions below will help you get started if you'd like to give this a try.

But first a word of warning: **this feature is experimental**. As far as I know it should be stable, but there are known security concerns, specifically with mounting ext2/3/4 volumes[^1].  It should only be enabled in trusted environments where potentially malicious users do not have shell access to your system.

# Requirements

In order to use this feature, you will need a 4.4.0-6.21 or later kernel in Ubuntu xenial. To follow these instructions you will also need to have lxd installed (help for this can be found [here](https://linuxcontainers.org/lxd/getting-started-cli/)).

# Setup

The first thing you need to do is flip the module parameters to enable user namespace mounts for fuse and/or ext4.

{% highlight console %}
$ echo Y | sudo tee /sys/module/fuse/parameters/userns_mounts
$ echo Y | sudo tee /sys/module/ext4/parameters/userns_mounts
{% endhighlight %}

A lxd container running under the default profile will not have the device nodes needed for mounting (/dev/fuse for fuse and some block device for ext4, e.g.  /dev/loop0) and will not be permitted to mount by AppArmor. We need to create a lxd profile which will include the device nodes and run the container without AppArmor confinement. This will be used on top of the lxd default profile.

{% highlight console %}
$ lxc profile create nsmount
$ lxc profile set nsmount raw.lxc lxc.aa_profile=unconfined
$ lxc profile device add nsmount fuse unix-char path=/dev/fuse
$ lxc profile device add nsmount loop0 unix-block path=/dev/loop0
{% endhighlight %}

Now create a new container using this profile.

{% highlight console %}
$ lxc launch ubuntu u1 -p default -p nsmount  
{% endhighlight %}

# Testing it out

Let's try fuse first. Launch a shell in your new container:

{% highlight console %}
$ lxc exec u1 -- /bin/bash
{% endhighlight %}

Inside the container we can create an ext2 filesystem and mount it using fuseext2, adding the '-o force' option to make the mount read/write.

{% highlight console %}
# apt-get install fuseext2
# dd if=/dev/zero of=ext2.img bs=1M count=8
# mkfs.ext2 ext2.img
# mkdir -p mount
# fuseext2 ext2.img mount -o force  
{% endhighlight %}

You can now browse and modify the filesystem. To unmount run:

{% highlight console %}
# fusermount -u mount
{% endhighlight %}

For ext4 we'll create the filesystem outside of the container and set this as the backing store for /dev/loop0.  If loop0 is already in use you can pick another loop device, in which case you'll also need to modify the nsmout lxd profile to make that device available in the container.

Back in the host run:

{% highlight console %}
$ dd if=/dev/zero of=ext4.img bs=1M count=8
$ mkfs.ext4 ext4.img
$ sudo losetup /dev/loop0 ext4.img  
{% endhighlight %}

Then back in the container run:

{% highlight console %}
# mkdir -p mount
# mount /dev/loop0 mount  
{% endhighlight %}

This filesystem can be unmounted in the usual way.

---

[^1]: The known security concern is that a malicious user could provide a crafted filesystem to exploit bugs in the in-kernel filesystem parsing code, or the user could change the backing store at runtime. Either of these could lead to kernel crashes or worse.
