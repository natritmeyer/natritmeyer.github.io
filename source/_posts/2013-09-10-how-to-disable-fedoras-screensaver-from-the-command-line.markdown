---
layout: post
title: "How to disable Fedora's screensaver from the command line"
date: 2013-09-10 17:56
comments: false
categories: devops fedora gnome vm
---
Today I found myself building a Fedora-based VM for running some tests that require chrome. I needed
to be able to prevent the screensaver from starting, and after a bit of googling I figured out how
to do it. Here's the command you need to run:

```sh
$ gsettings set org.gnome.desktop.session idle-delay 0
```

I tested this on Fedora 19 but I guess it should work on any Gnome 3 distro...

Hope that helps!
