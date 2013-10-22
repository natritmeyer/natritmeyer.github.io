---
layout: post
title: "XPath Queries in Chrome"
date: 2013-10-22 20:31
comments: false
categories: chrome xpath devtools
---

Running xpath queries shouldn't be hard. It used to be that you'd have to install
plugins into whatever browser you were using. They were often clumsy and always buggy.
And, even though querying using CSS selectors has grown more popular in the
automated-acceptance-web-test world, there are still sometimes that xpath is the only
option.

It turns out that Chrome has built in support for xpath queries in its dev tools. Simply
run the following in the console tab:

`$x("your_xpath_here")`

For example, run the following command in the console when the `http://www.google.co.uk`
page is loaded:

`$x("//input[@name='q']")`

Here's a screenshot of `$x("your_xpath")` in action:

<img src="{{ root_url }}/images/chrome_xpath.png" />

Hope that helps!

