---
layout: post
title:  "Solving UI Lag in Yosemite"
date:   2015-01-24 15:18:22 +0100
categories: posts
---

After using Yosemite for a couple of hours on end, the system usually gets very slow. Especially User Interface transitions and animations (such as when opening Expose) become choppy and take longer to perform than they should. It seems that the Window Server is the cause for this problem, as it shows very high CPU activity whenever the problem occurs.

I used to resolve this issue with a reboot, but this is not really convenient. It turns out that simply killing the `WindowServer` process from the command line is sufficient:

{% highlight bash %}
sudo pkill -9 "WindowServer"
{% endhighlight %}

The system will bounce you back to the login screen and after logging in everything will be running smoothly again.
