---
layout: post
title: Interacting with EPFL's Tequila in Python
categories: Programming
---

The Swiss Federal Instituts for Technology (EPFL) uses a centralized login system for all its web application, called Tequila.
Since I sometimes write webscrapers for some school services (with the help of *@gcmalloc*), I created a simple python package to handle authentification.

This package is now available on the Python Package Index, so you can get it by simply running `pip install tequila-sessions`
(You probably want to do it inside a virtualenv).
This module only export one functions `create_tequila_session`, which can be used as follows:

{% highlight python %}
#Prings the logged-in homepage of Moodle
import tequila
import getpass

USER = "GASPAR"

session = tequila.create_tequila_session(USER, getpass.getpass())
page = session.get("http://moodle.epfl.ch").text
print(page)
{% endhighlight %}

Hope some of you find this useful!


