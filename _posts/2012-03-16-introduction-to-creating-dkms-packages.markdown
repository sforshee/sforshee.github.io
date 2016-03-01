---
layout: post
title: Introduction to Creating DKMS Packages for Ubuntu, Part 1
date: '2012-03-16T09:00:00.000-05:00'
author: Seth Forshee
tags:
- Linux
- dkms
modified_time: '2012-03-16T09:03:05.330-05:00'
blogger_id: tag:blogger.com,1999:blog-8501269611012488187.post-2044436290227594305
blogger_orig_url: http://blog.forshee.me/2012/03/introduction-to-creating-dkms-packages.html
---

Dynamic Kernel Module Support (DKMS) is a really useful framework that allows kernel modules to be built dynamically for each kernel present on a system. The modules can also be automatically rebuilt any time a new kernel is installed. This is obviously a great tool if you want to distribute a driver that isn't included in the Linux kernel. I've used it a number of times to distribute test versions of drivers that are still under development.

It turns out that using DKMS is actually pretty easy, most of the time anyway. In this post I'm going to cover the basics of setting up a DKMS build on your local machine and the quick-and-dirty method of creating a debian package so that others can easily install the driver on their systems. Then in the next post I'll go over a slightly more sophisticated technique for using DKMS with debian packaging.

For this explanation I'm going to go through an example setup with a driver I've been working on recently to support the gmux device on some Apple laptops. This device is responsible for switching between the two GPUs found in these systems as well as backlight control. The files used in this example can be downloaded [here](http://zinc.canonical.com/~sforshee/dkms-demo/apple-gmux-dkms-0.1.tar.gz). This archive should be extracted to the /usr/src directory. You'll also need to install the DKMS tools, which in Ubuntu is done by simply installing the package named "dkms."

DKMS expects the source for packages to be located under `/usr/src` in a directory named according to the format `<module>-<version>`. In this case, my module is apple-gmux and I'm giving it the version 0.1, so the source files are located in `/usr/src/apple-gmux-0.1`. Within this directory is configuration file named dkms.conf. In the case of apple-gmux this file is very simple. Let's go through it line by line.

{% highlight text linenos %}
PACKAGE_NAME="apple-gmux"
PACKAGE_VERSION="0.1"

BUILT_MODULE_NAME="apple-gmux"
DEST_MODULE_LOCATION="/updates"
AUTOINSTALL="yes"
{% endhighlight %}

- Line #1: `PACKAGE_NAME` unsurprisingly defines the name of the package. This needs to match the module name used in naming the directory.
- Line #2: `PACKAGE_VERSION` tells DKMS the current version number, which once again must match the one used in the path.
- Line #4: `BUILT_MODULE_NAME` tells DKMS about the filename of the module(s) that will be built. In this case the module is named apple-gmux.ko. This variable is actually an array, so if your package supplies multiple modules these can be specified as `BUILT_MODULE_NAME[0]`, etc.
- Line #5: `DEST_MODULE_LOCATION` tells DKMS where to install the module(s) relative to the kernel's module install path (typically /lib/modules/$(uname -r)). This is also an array, with each entry specifying the install path of the corresponding entry in `BUILT_MODULE_NAME`. For Ubuntu at least it's a good idea to install to /updates, as modules here automatically override the ones that shipped with your kernel.
- Line #6: `AUTOINSTALL`, when set to yes, directs DKMS to automatically install the module(s) for any new kernels you install.

There are a number of other options, all documented in the [DKMS man page](http://manpages.ubuntu.com/manpages/precise/en/man8/dkms.8.html).

Besides dkms.conf we also have the source file for the module as well as a makefile. The makefile is in the kernel format and has access to the kernel configuration variables. In this example the source is a single file, but it can be more complicated. You can have a source tree with multiple directories and multiple makefiles just like in the kernel. DKMS will use kbuild to invoke the top-level makefile, which can descend into subdirectories in the usual fashion. But since apple-gmux is so simple, I've opted to place everything in the top-level directory.

Once the DKMS module's source tree is in place, it's time to build the module.  Working with DKMS packages is done using the `dkms` program by invoking a series of commands with the following syntax.

{% highlight console %}
$ dkms <command> <module>/<version>
{% endhighlight %}

To build and install apple-gmux, we must invoke the `add` command followed by the `build` and `install` commands. Note that this only builds and installs for the current kernel; the `build` and `install` commands must be invoked with the -k option for other kernels you have installed.

{% highlight console %}
$ sudo dkms add apple-gmux/0.1  
$ sudo dkms build apple-gmux/0.1  
$ sudo dkms install apple-gmux/0.1  
{% endhighlight %}

Now we're able to use modprobe to load the apple-gmux module just like any other module. If we want to distribute the module for others to use, DKMS even has `mkdeb` and `mkrpm` commands do automatically create a Debian or RPM package.

This all works great for one-off DKMS packages, but it's a little cumbersome for something that will be updated regularly. In [part 2](http://blog.forshee.me/2012/03/introduction-to-creating-dkms-packages_16.html) we'll cover a technique to simplify matters for frequently updated packages.
