---
layout: post
title: "A script to install chromedriver on linux"
date: 2013-09-11 17:48
comments: false
categories: devops chromedriver linux chrome browser_testing
---
Installing [chromedriver](https://code.google.com/p/chromedriver/) on linux can be annoying.
Here's a script that takes the pain out of installing the [latest](https://code.google.com/p/chromedriver/downloads/list)
(at time of writing!) 64 bit version of the driver:

```sh
#!/bin/bash
wget https://chromedriver.googlecode.com/files/chromedriver_linux64_2.3.zip
unzip chromedriver_linux64_2.3.zip
sudo cp chromedriver /usr/bin/chromedriver
sudo chown root /usr/bin/chromedriver
sudo chmod +x /usr/bin/chromedriver
sudo chmod 755 /usr/bin/chromedriver
```

I've tested it on Fedora 19.

If you need a different version of the driver, just change the link on the first line of the script.

Hope that helps!
