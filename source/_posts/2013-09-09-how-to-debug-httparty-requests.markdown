---
layout: post
title: "How to debug HTTParty requests"
date: 2013-09-09 19:42
comments: false
categories: ruby, http, httparty
---

I very frequently find myself debugging http calls. Curl makes it easy to do this through its `-v` switch that lets you see exactly what it's doing. For example:

```sh
$ curl -v http://www.google.co.uk
* About to connect() to www.google.co.uk port 80 (#0)
*   Trying 173.194.41.95... connected
* Connected to www.google.co.uk (173.194.41.95) port 80 (#0)
> GET / HTTP/1.1
> User-Agent: curl/7.21.4 (universal-apple-darwin11.0) libcurl/7.21.4 OpenSSL/0.9.8x zlib/1.2.5
> Host: www.google.co.uk
> Accept: */*

```

Hostname-to-IP resolution, any SSL handshaking as well as full header details are on display. If you're doing a
POST or PUT you get the body too. All very helpful.

But, once I've figured out what I need my code to do I need to translate my curl incantations into [HTTParty](http://johnnunemaker.com/httparty/) - currently my favourite ruby http library. It is possible to get
similar details out of httparty but it's a little esoteric.

Here's a fairly representative use of httparty:

```ruby
require 'httparty'

class Google
  include HTTParty
  
  base_uri "http://www.google.co.uk"
  
  def home_page
    self.class.get "/"
  end
end

goog = Google.new
puts goog.home_page
```

But if I run that, I don't get any debug info (as expected):

```sh
$ ruby httparty_debug.rb 
<!doctype html><html itemscope="" itemtype="http://schema.org/WebPage"><head><meta ...
...
```

No headers, no hostname-to-ip resolution, no SSL details.

To get the debug info, add `debug_output $stdout` on a new line after `include HTTParty`, eg:

```ruby
class Google
  include HTTParty
  debug_output $stdout # <= this is it!
  
  #...
end
```

The result of running that will be a console filled with debug info!

```sh
$ ruby httparty_debug.rb 
opening connection to www.google.co.uk:80...
opened
<- "GET / HTTP/1.1\r\nConnection: close\r\nHost: www.google.co.uk\r\n\r\n"
-> "HTTP/1.1 200 OK\r\n"
-> "Date: Mon, 09 Sep 2013 18:59:12 GMT\r\n"
-> "Expires: -1\r\n"
...
...
...
```

Note that if you'd rather the debug info went to `$stderr`, change `debug_output $stdout` to `debug_output $stderr`.

Hope that helps!

