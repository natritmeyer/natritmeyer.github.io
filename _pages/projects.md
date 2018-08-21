---
title: "Projects"
permalink: /projects
layout: single
author_profile: true
read_time: false
toc: true
---

A quick summary of some of my open source projects.

# Ruby

## [site-prism](https://github.com/natritmeyer/site_prism)

* _A Page Object Model DSL for Capybara_
* Itch scratched: Capybara interacted with browsers reliably, but its API didn't lend itself to the page object model. So I wrote a thin wrapper around capybara to allow it to be used with the page object model.
* Currently under active development, managed by [Luke Hill](https://github.com/luke-hill) - he's doing a fine job of it too
* Downloaded [> 2 million times](https://rubygems.org/gems/site_prism)

## [yarjuf](https://github.com/natritmeyer/yarjuf)

* _Yet Another RSpec JUnit Formatter_
* Itch scratched: At the time there wasn't an rspec plugin that would reliably generate junit-formatted xml. So I wrote yet another rspec junit formatter, which seemed an appropriate name.
* No longer under active development
* Downloaded [> 1 million times](https://rubygems.org/gems/yarjuf)

## [bewildr](https://github.com/natritmeyer/bewildr)

* _An [IronRuby](http://ironruby.net)(!) DSL around [Microsoft's UI Automation API](https://docs.microsoft.com/en-us/windows/desktop/winauto/entry-uiauto-win32)_
* Itch scratched: Due to a project mandate the tests for the journalists' CMS behind the BBC News website had to be tested with ruby. Nothing in the ruby ecosystem existed that could drive the .Net WPF application so I wrote Bewildr.
* No longer under active development
* Downloaded [~ 40k times](https://rubygems.org/gems/bewildr) - who on earth was using it other than my team I will never know...
