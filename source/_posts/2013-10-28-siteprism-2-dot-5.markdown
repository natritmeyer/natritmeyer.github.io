---
layout: post
title: "SitePrism 2.5"
date: 2013-10-28 18:53
comments: false
categories: site_prism page_object_model ruby gems test_automation browser_testing
---

SitePrism 2.5 is made up mainly of contributions from various people. From the [HISTORY](https://github.com/natritmeyer/site_prism/blob/master/HISTORY) file:

* added ability to select iframe by index - thanks to [Mike Kelly](https://github.com/mikekelly)
* site_prism now does lazy loading - thanks to [MrSutter](https://github.com/mrsutter)
* added config block and improved capybara integration - thanks to [tmertens](https://github.com/tmertens) (and to [LukasMac](https://github.com/LukasMac) for testing it)
* changed #set_url to convert its input to a string - thanks to [Jared Fraser](https://github.com/modsognir)

Due to various lame excuses I've taken ages to get this version out so please forgive me.

To get an idea of the main functional changes, see the following sections of the [ReadMe](https://github.com/natritmeyer/site_prism/blob/master/README.md):

* [SitePrism configuration to use capybara's implicit waits](https://github.com/natritmeyer/site_prism#siteprism-configuration)
```ruby
SitePrism.configure do |config|
  config.use_implicit_waits = true
end
```
* [Ability to use capybara's query options in page object definitions and assertions](https://github.com/natritmeyer/site_prism#using-capybara-query-options)
```ruby
class SearchResults < SitePrism::Page
  element :view_more, "li", text: "View More"
end

@results_page.<element_or_section_name> :text => "Welcome!"
@results_page.has_<element_or_section_name>? :count => 25
@results_page.has_no_<element_or_section_name>? :text => "Logout"
@results_page.wait_for_<element_or_section_name> :count => 25
@results_page.wait_until_<element_or_section_name>_visible :text => "Some ajaxy text appears!"
@results_page.wait_until_<element_or_section_name>_invisible :text => "Some ajaxy text disappears!"
```

Both changes supplied by [tmertens](https://github.com/tmertens), both changes being the last 2 gripes I hear from people about SitePrism :)

Thanks to [LukasMac](https://github.com/LukasMac) for his extensive testing of this version.

Finally, add your project or company to the [Who is using SitePrism](https://github.com/natritmeyer/site_prism/wiki/Who-is-using-SitePrism) page. It would make my day!

Hope that helps!

