---
layout: post
title: Using ftrace to Identify the Process Calling a Kernel Function
date: '2011-12-15T17:52:00.001-06:00'
author: Seth Forshee
tags:
- Linux
modified_time: '2011-12-16T10:00:08.048-06:00'
blogger_id: tag:blogger.com,1999:blog-8501269611012488187.post-6568643671275466181
blogger_orig_url: http://blog.forshee.me/2011/12/using-ftrace-to-identify-process.html
---

Sometimes it's useful to know the context from which a kernel function is being called. For instance, I recently wanted to know what process was responsible for adjusting the keyboard backlight in response to hotkey presses on the MacBook Air I've been working with. Based on the way the LED class driver works, I knew that the callback into the hardware-specific driver (in this case applesmc_brightness_set() from drivers/hwmon/applesmc.c) would be called from the context of the process performing the backlight adjustment.

The easiest way that I know of to do this is using the ftrace function tracer, which will display the name and process ID for the current task each kernel function that is executed. Even better, if CONFIG_DYNAMIC_FTRACE is enabled it the kernel build configuration (as it is in the stock Ubuntu kernels), ftrace can be limited to tracing only a single function. Here's how to do it.

The files used for working with ftrace are located in the /sys/kernel/debug/tracing directory. All commands that follow assume that this is the current working directory, and all must be executed as root.

In order to limit the trace to only applesmc_brightness_set(), I used the *set_ftrace_filter* file.

{% highlight console %}
# echo applesmc_brightness_set > set_ftrace_filter
{% endhighlight %}

ftrace has a number of filters for different propose. As I mentioned earlier, the one needed to trace function calls is predictably named the *function* tracer.

{% highlight console %}
# echo function > current_tracer
{% endhighlight %}

With those command ftrace was set up. The next step was to enable tracing, adjust the keyboard backlight using the hotkey, and disable tracing.

{% highlight console %}
# echo 1 > tracing_on
# # adjust backlight...
# echo 0 > tracing on
{% endhighlight %}

The results of the trace a read from the *trace* file.

{% highlight text %}
# cat trace
# tracer: function
#
#           TASK-PID    CPU#    TIMESTAMP  FUNCTION
#              | |       |          |         |
         upowerd-1264  [003]  5875.189679: applesmc_brightness_set <-led_brightness_store
{% endhighlight %}

Using this technique, I was quickly able to determine that the upowerd processes is responsible for changing the keyboard backlight whenever I press the hotkey.
