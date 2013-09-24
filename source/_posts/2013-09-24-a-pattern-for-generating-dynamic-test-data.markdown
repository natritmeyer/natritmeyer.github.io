---
layout: post
title: "A pattern for generating dynamic test data"
date: 2013-09-24 20:20
comments: false
categories: test_data patterns reliable_tests 
---

When writing acceptance tests that use test data (and that's most of them), I like to deal with abstractions of that data rather than
the data itself. The reasons for this are:

* Using abstractions leads to more expressive tests: `@expired_account` instead of `77481` better relays the intent of the test to the reader
* Using abstractions frees me from worrying about the details of the test data - no hard coded IDs!
* Lack of hard coded data means that when something needs to change I only need to change it in one place, leading to more robust and maintainable code
* Test code is code - it should adhere to all the usual best practices for coding; abstraction is one of them

To illustrate this let's compare the following two chunks of code:

```ruby
@bob = Human.new
AccountService.create_account_for @bob
AccountService.should have_account_for @bob
```

...and...

```ruby
@name = "Bob"
@age = 50
@country = "Botswana"

AccountService.create_account_for @name, @age, @country

xml = AccountService.account_details_for_user_with_details @name, @age, @country
xml.at_xpath("//account/name").text.should == @name
xml.at_xpath("//account/name").text.should == @age
xml.at_xpath("//account/name").text.should == @country
```

Which one more quickly and clearly transmits the intent of the test to the reader? I'd argue that the first does. The reader is not distracted with unnecessary details; instead they know they are creating a new generic `Human` object, creating an account with it and then verifying that the account has been created. The second one achieves the same thing but uses a lot more code - figuring out the intent of the test takes more time and effort; the result is less maintainable too.

It doesn't take much work to use test data abstractions like `Human` and what little work is required is paid back many, many, many times over. Eg: creating the above `Human` class is as simple as this:

```ruby
require 'builder'

class Human
  attr_accessor :name
  attr_accessor :age
  attr_accessor :country
  
  def initialize
    @name = "Bob"
    @age = 50
    @country = "Botswana"
  end
  
  def to_xml
    b = Builder::XmlMarkup.new :indent => 2
    b.instruct! :xml, :version => "1.0", :encoding => "utf-8"
    b.Human do
      b.name @name
      b.age @age
      b.country @country
    end
    b.target!
  end
end
```

Let's take a look at what's going on.

The `initialize` method creates sensible default test data attributes when an instance of the `Human` class is created. The test data attributes are exposed using `attr_accessor`s so the test data object can be changed in the test. The `to_xml` method creates an XML representation of the human. This could just as well be a `to_json` method that spits out a json representation of the human*.

<sub>
  * For those who take abstraction seriously this is a fine place to use the Template or Strategy patterns to decide between json and xml output at runtime.
</sub>

Being able to create objects containing default test data that can be changed in the test will lead to more expressive test code (have I said that already?). Here's what I mean:

```ruby
@baby = Human.new
@baby.age = 1
```

The defaults, other than `age` are sensible, so we'll leave them. The only one we need to change is `age`, so we override the default value with `1`. Now that the `@baby` instance of `Human` has been created, when we see `@baby` in the test code it will read nicely:

```ruby
AccountService.create_account_for(@baby).should == :too_young
```

But still, it would be nice to not have to change the age of `@baby` in the test - why can't this happen automatically?

Well, by using the [Factory pattern](https://en.wikipedia.org/wiki/Factory_method_pattern) you can create specific instances of test data without cluttering up your test code. Factory classes are those that create instances of other classes, hiding any complicated setup. Eg:

```ruby
class HumanFactory
  def self.standard
    Human.new
  end

  def baby
    human = Human.new
    human.age = 1
    human
  end

  def self.too_old
    human = Human.new
    human.age = 500
    human.name = "Methuselah"
    human
  end
end

#and to use the factory...

@standard = HumanFactory.standard
@baby = HumanFactory.baby
@geriatric = HumanFactory.too_old
```

Thus far we are able to dynamically create objects that represent test data.

There is one more important thing in this pattern - the separation between the data and the representation of the data. When we call `to_xml`, we get back a string containing an XML representation of the test data object. What's this for? Well, in your tests you can use the output of the method to pass to services, etc - that's what the `AccountService.create_account_for(@baby)` example is doing - the `to_xml` method would be called inside the `create_account_for` method.

An essential attribute of the `to_xml` method is that however many times it is called, unless the data changes it should always return the same thing. For this reason the following would be bad:

```ruby
require 'builder'
require 'active_support/time'

class Human
  attr_accessor :birthday
  
  def initialize
    #@birthday not set to sensible default :(
  end
  
  def to_xml
    b = Builder::XmlMarkup.new :indent => 2
    b.instruct! :xml, :version => "1.0", :encoding => "utf-8"
    b.Human do
      b.birthday Time.now # <-- this is bad!
    end
    b.target!
  end
end
```

The problem with the above is that in the `to_xml` method there is a call to something that will change every time it is called; `Time.now`. To illustrate the point here's what happens when you create an instance of the above class and call `to_xml` on it lots of times:

```text
irb(main):030:0> puts bob.to_xml
<?xml version="1.0" encoding="utf-8"?>
<Human>
  <birthday>2013-09-24 21:35:14 +0100</birthday>
</Human>
=> nil
irb(main):031:0> puts bob.to_xml
<?xml version="1.0" encoding="utf-8"?>
<Human>
  <birthday>2013-09-24 21:35:18 +0100</birthday>
</Human>
=> nil
irb(main):032:0> puts bob.to_xml
<?xml version="1.0" encoding="utf-8"?>
<Human>
  <birthday>2013-09-24 21:35:25 +0100</birthday>
</Human>
=> nil
```

As you can see, every time we call `bob.to_xml` his birthday changes. Not great. Instead, the `to_xml` code should be changed so that all it has to worry about is *presenting* the data. Here's how to do it:

```ruby
require 'builder'

class Human
  attr_accessor :birthday
  
  def initialize
    @birthday = Time.now
  end
  
  def to_xml
    b = Builder::XmlMarkup.new :indent => 2
    b.instruct! :xml, :version => "1.0", :encoding => "utf-8"
    b.Human do
      b.birthday @birthday.strftime "%Y-%m-%d"
    end
    b.target!
  end
end
```

This time, when we call `to_xml` a number of times, we'll see that as well as the data being static, it is also now presented correctly:

```text
irb(main):054:0> bob = Human.new
=> #<Human:0x007fc1c1d44d80 @birthday=2013-09-24 21:38:19 +0100>
irb(main):055:0> puts bob.to_xml
<?xml version="1.0" encoding="utf-8"?>
<Human>
  <birthday>2013-09-24</birthday>
</Human>
=> nil
irb(main):056:0> puts bob.to_xml
<?xml version="1.0" encoding="utf-8"?>
<Human>
  <birthday>2013-09-24</birthday>
</Human>
=> nil
irb(main):057:0> puts bob.to_xml
<?xml version="1.0" encoding="utf-8"?>
<Human>
  <birthday>2013-09-24</birthday>
</Human>
=> nil
```

Summary:

1. Create test data classes that represent types of test data you use in your acceptance tests (eg: `class Human`)
1. Expose attributes of those test data classes using accessors (eg: `attr_accessor :name`)
1. Set any attributes in the test data class to sensible defaults in the `initialize` method
1. Create `to_xml`/`to_json`/`to_csv`/etc rendering methods that will render the test data object in the formats required by your system under test
1. Ensure that rendering methods do not include any logic other than presentation logic
1. Use your test data classes in your test code
1. \#win

Hope that helps!