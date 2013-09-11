---
layout: post
title: "Libs required for ruby dev on Fedora 19"
date: 2013-09-10 18:12
comments: false
categories: devops linux fedora ruby
---
In setting up a [Fedora 19](https://fedoraproject.org/) VM for web testing with [Chrome](https://www.google.com/chrome)
I found that I needed to install a few packages before things started to work. After some trial-and-error, I came up
with the following command:

```sh
$ sudo yum install -y git vim gcc gcc-c++ ruby ruby-devel libxml2 libxml2-devel libxslt libxslt-devel
```

It should install everything you need to start coding in ruby (the above command assumes your code is stored in git and you're
using vim) - I had particular trouble trying to get [nokogiri](http://nokogiri.org/) to work. But the above command solved it :)

Hope that helps!
