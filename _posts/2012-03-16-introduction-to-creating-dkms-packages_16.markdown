---
layout: post
title: Introduction to Creating DKMS Packages for Ubuntu, Part 2
date: '2012-03-16T09:02:00.000-05:00'
author: Seth Forshee
tags:
- Linux
- dkms
modified_time: '2012-03-16T09:02:13.953-05:00'
blogger_id: tag:blogger.com,1999:blog-8501269611012488187.post-7311014127375457319
blogger_orig_url: http://blog.forshee.me/2012/03/introduction-to-creating-dkms-packages_16.html
redirect_from: "/2012/03/introduction-to-creating-dkms-packages_16.html"
---

The DKMS framework makes it easy to create and distribute an out-of-tree kernel module, as demonstrated in [part 1](http://blog.forshee.me/2012/03/introduction-to-creating-dkms-packages.html). However, the process is rather manual (read: error prone) for packages that will be updated regularly. This post is going to look at a way to set up the debian packaging components to make updating easier. A basic familiarity with debian packaging is assumed.

As an example, we'll use an updated version of the apple-gmux-dkms source tree from part 1, which can be donwloaded [here](http://kernel.ubuntu.com/~sforshee/dkms-demo/apple-gmux-dkms-0.2.tar.gz). This time it's not necessary to extract it to `/usr/src`; just extract it to wherever you like in your home directory.

The layout of the files has changed a little bit. The source file and makefile are still in the root of the tree, but `dkms.conf` has been moved into the `debian` directory and renamed to `apple-gmux-dkms.dkms.in`. We'll talk more about this file later. Most of the remaining files are the typical debian packaging boilerplate; examining these files is left as an exercise for the reader. The one file we will take a closer look at is `debian/rules`.

The first item of note in the rules file is that it grabs the version number for the package from the changelog with the following.

{% highlight make %}
version := $(shell dpkg-parsechangelog | grep '^Version:' | cut -d' ' -f2 | cut -d- -f1)  
{% endhighlight %}

This eliminates the error-prone process of making sure the version gets updated throughout the package by making the changelog the canonical location for the version number. Towards that end, the rules file also has a rule to auto-generate the DKMS configuration file, `debian/apple-gmux-dkms.dkms`, from `apple-gmux-dkms.dkms.in`. This file is identical to the `dkms.conf` file used in part 1, except that the version number has been replaced by the string `@VERSION@`. A sed command is used to generate the config file with `@VERSION@` replaced by the actual version number.

{% highlight make %}
debian/apple-gmux-dkms.dkms: debian/apple-gmux-dkms.dkms.in  
    sed s/@VERSION@/$(version)/g $< > $@  
{% endhighlight %}

The other item worth noting is that there's a DKMS debhelper, dh_dkms, which makes setting up the rules file a breeze when used along with the dh helper (for more information about the dh helper consult the [man page](http://manpages.ubuntu.com/manpages/precise/man1/dh.1.html)). We just need a rule to let dh do its thing:

{% highlight make %}
%:  
    dh $@  
{% endhighlight %}

Then we need to override a few of the debhelper commands to install files for the DKMS package and to invoke dh_dkms. We also need to clean up the auto-generated DKMS configuration file, and since we're not actually building anything to generate the package we override the build command to do nothing. This leaves us with the following override rules.

{% highlight make %}
override_dh_clean:
        dh_clean debian/apple-gmux-dkms.dkms

override_dh_auto_build:
        :

override_dh_auto_install: debian/apple-gmux-dkms.dkms
        dh_install apple-gmux.c Makefile /usr/src/apple-gmux-$(version)/
        dh_dkms

override_dh_auto_install_indep: debian/apple-gmux-dkms.dkms
{% endhighlight %}

That's it! Now you can build the binary and source packages with the usual debian packaging commands. When it comes time to update the package, you only need to update the version number in the changelog to get it updated throughout the package.
