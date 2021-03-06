- What is Cuba?
- Installation
- Hola Cuba
- Using Cuba in a classic fashion
- Templates with Tilt
- Cuba's love affair with Mote
- Helpers
- Organizing your application
- Using middleware
- Testing
- Shotgun
- Deployment
- Authoring your own plugins
- Cuba Contrib
- Designing an API

# What is Cuba?

Cuba is a microframework for web development originally inspired
by [Rum][rum], a tiny but powerful mapper for [Rack][rack]
applications.

It integrates many templating engines via [Tilt][tilt], it's
very easy to test with either [rack-test][rack-test] or
[Capybara][capybara], and it's extensible with plugins.

# Installation

Install the gem with the following command:

    $ gem install cuba

# Hola Cuba

Open up your text editor, and write the following lines:

```ruby
require 'cuba'

Cuba.define do
  on root do
    res.write "Hello world"
  end
end

run Cuba
```

Save this file as `config.ru`. You can now try running it:

```term
$ rackup config.ru
```

Going to http://localhost:9292/ should say the proverbial "Hello
world".

## What's happening?

The fastest way to define a Cuba app is, as you might have
guessed, with the `Cuba.define do ... end` block. Within that
block, you have some objects available, such as `env`, `req` and
`res`, and a very useful function, `on`. Let's talk about `on`
first.

### Flow control with `on`

If you know how to deal with nested `if`/`else` or nested `case`
blocks, you will get the idea behind `on` pretty easily.

It executes the passed block only if all of its arguments evaluate
to `true`. Let's look at this example:

```ruby
require 'cuba'

Cuba.define do
  on true, 2 + 2 == 4 do
    res.write "Hello world"
  end
end

run Cuba
```

As both arguments (`true` and `2 + 2 == 4`) evaluate to `true`,
the block is executed and the string `"Hello world"` is written to
the response object. You are not limited to having one `on` block,
though. Let's look at another example:

```ruby
require 'cuba'

Cuba.define do
  on false do
    res.write "This block will never execute"
  end

  on true do
    res.write "This block will execute"
  end
end

run Cuba
```

When one of the arguments to `on` evaluates to `false` or `nil`,
the block is skipped and the next call to `on` takes place. In
this case, only the last block will run. One last example will
clarify what happens when more than one call to `on` succeeds:

```ruby
require 'cuba'

Cuba.define do
  on true do
    res.write "This block will execute"
  end

  on true do
    res.write "This block will never execute"
  end
end

run Cuba
```

In this case, the first successful call to `on` wins. Once the
attached block is executed, the application will never evaluate
subsequent calls to `on`. This behavior is very important, because
not only it is key to Cuba's performance, but it can also be a
huge gotcha if you ignore it.

## Nested `on` blocks

What happens when you nest several `on` blocks is what you would
expect from nested `if`/`else` statements. Why don't we just use
`if` and `else`, you may ask: the answer is that `on` does a bit
more, and we will learn about that later.

```ruby
require 'cuba'

Cuba.define do
  on true do
    on true do
      on true do
        res.write "Hello from a deeply nested block"
      end
    end
  end
end

run Cuba
```

All the blocks will get executed, and the response will contain
the passed string.

# Using `env`

The `env` object contains all the information of the request in
its raw form. All the CGI environment variables are present and
accessible:

```ruby
# Display the HTTP verb
res.write env["HTTP_REQUEST_METHOD"]
```

You can inspect the contents of the `env` variable directly, but
usually it's easier to access its values with the `req` object,
which is just a bit more than a wrapper with some syntax sugar.

# Using `req`

The `req` objects provides helpers and shortcuts for accessing the
request environment variables. For example, if you want to display
the request method as in the previous section, you can instead
call `req.request_method`. Alternatively, you can ask whether a
request was a POST with the `req.post?` helper. Here's a short
list of some methods provided by the `req` object:

```ruby
# Inspecting the request method
req.delete?
req.get?
req.head?
req.options?
req.patch?
req.post?
req.put?
req.trace?

# XMLHttpRequest verification
req.xhr?

# Accessing the query string
req.query_string

# Accessing the passed parameters
req.params

# Accessing the user agent
req.user_agent
```

You can read the documentation for `Rack::Request` for more
information about the methods available and how to use them.

# Using `res`

In the previous examples, we have used `res` to write strings
that are returned to the browser. It is an instance of
`Cuba::Response`, an object that defines methods for setting the
response status, the headers and the body.

## Default values

The default status of the `res` instance is 200, but if no routes
match the request it is changed to 404 to denote that no resource
was found. You can override its value:

```ruby
# Setting a different status
res.status = 206
```

The default hash of headers is:

```ruby
{ "Content-Type" => "text/html; charset=utf-8" })
```

You can add, modify or remove headers by accessing the hash:

```ruby
res.headers["Content-Type"] = "application/json"
```

The default body is an empty array. When you write to the `res`
object, the strings are appended to `res.body` and a header with
the length of the response is calculated. Multiple calls to
`res.write` are possible.

```ruby
res.write "Hello, "
res.write "world"
```

## Redirecting

The `res` object also provides a helper for HTTP redirection.

```ruby
res.redirect "http://example.com"
```

It sets the passed value to the "Location" header and updates the
status to 302. An optional second parameter corresponds to the
status code.

## Cookies

You can add or remove cookies with these two methods:

```ruby
res.set_cookie "some-key", "some-value"
res.delete_cookie "some-key"
```

## Rack API

The Rack API expects an array with exactly three elements: a
status code, a hash of headers and an enumerable body. Cuba
generates this array by calling `res.finish`. If you want to check
how the response looks like at any point in time, you can print
the result of `res.finish`.

```ruby
# Render the current value of res.finish
res.write res.finish.inspect
```

# Using Cuba like most web frameworks

Typically in most frameworks, you're limited to routing using only
the path. Similarly, Cuba can be used in the same manner:

```ruby
Cuba.define do
  on get, "about" do
    res.write "About"
  end

  on get, "contact" do
    res.write "Contact"
  end

  on post, "contact" do
    res.write "Just posted info."
  end
end
```

# Templates with Tilt

Cuba ships with a plugin that provides helpers for rendering
templates. It uses [Tilt][tilt], a gem that interfaces with many
template engines.

``` ruby
require "cuba/render"

Cuba.plugin Cuba::Render

Cuba.define do
  on default do

    # Within the partial, you will have access to the local
    # variable `content`, that will hold the value "hello, world".
    res.write render("home.haml", content: "hello, world")
  end
end
```

Note that in order to use this plugin you need to have
[Tilt][tilt] installed, along with the templating engines you want
to use.

Two more helpers are available, but let's first shed some light to
the configuration options.

The `Cuba::Render` plugin looks for settings in the `:render`
namespace:

```ruby
# The default path to views is `File.expand_path("views", Dir.pwd)`.
Cuba.settings[:render].store(:views, File.expand_path("tpl", Dir.pwd))

# The default template engine is `erb`.
Cuba.settings[:render].store(:template_engine, "haml")
```

# Cuba's love afair with Mote

# Helpers

# Organizing your application

# Using middleware

# Testing (the light and easy way)

# Shotgun (if you think it's slow, you are doing something wrong)

# Deployment

## Heroku

## Deploying on your own server

# Authoring your own plugins

Cuba comes with a very simple, yet powerful plugin system.

## Adding a couple of helper methods

## A more complete plugin

# Cuba Contrib

Over the course of the years, we've been able to identify the most
basic set of features that we've needed for the majority of our
projects. We've extracted most of these in a plugin, so others can
benefit from our work.

## Getting cuba-contrib

The source code of cuba contrib is [hosted in github][cuba-contrib].
To install it simply use rubygems:

```term
$ gem install cuba-contrib
```

After that, you should be ready to go!

## Setting up your project to use cuba-contrib

Once you have cuba contrib installed, you can require it in one of
two ways:

1. Require everything - you do this by requiring `cuba/contrib`.
2. Require only specific plugins - you can cherry pick only the
   plugins you want, e.g. `cuba/prelude`, `cuba/text_helpers`.

The following example briefly shows a Cuba app making use of
`Cuba::Prelude`.

```ruby
require 'cuba'
require 'cuba/prelude'

Cuba.plugin Cuba::Prelude

Cuba.define do
  on root do
    # this writes http%3A%2F%2Fwww.google.com
    res.write urlencode("http://www.google.com.com")

    # writes Cuba &amp; Cuba Contrib
    res.write h("Cuba & Cuba Contrib")
  end
end
```

### Text Helpers

We've all need these functions time and again, and we've written
them as simply as we possibly could. There was rarely a project
that didn't use all of these helpers.

```ruby
require 'cuba'
require 'cuba/text_helpers'

Cuba.plugin Cuba::TextHelpers

Cuba.define do
  on root do
    lorem = "Lorem ipsum dolor sit amet, consectetur adipisicing " +
            "elit, sed do eiusmod tempor incididunt ut labore et " +
            "dolore magna aliqua."

    res.write truncate(lorem, 10)
    # => Lorem ipsu...

    res.write titlecase("hello there")
    # => Hello There

    res.write currency(99.9)
    # => $ 99.90
    res.write currency(99.9, "EUR")
    # => EUR 99.90

    reply = <<-EOT
    Greetings,

    This is my response to you.

    Thanks,
    john
    EOT

    res.write nl2br(reply)
    # => Greetings,<br><br>This is my response to
    # you.<br><br>Thanks,<br>john

    res.write delimit(100_000)
    # => 100,000
    res.write delimit(1_000_000, ".")
    # => 1.000.000

    res.write markdown("# Title\n## Subtitle")
    # => <h1>Subtitle</h1> <h2>Subtitle</h2>
  end
end
```

### Storing variables using Cuba::With

Often times when you compose your apps, you run into the problem
of needing to stash certain variables. Good thing for us, there
has been a standard pattern to do this with rack apps, which is to
store store in the `env`.

```ruby
on "users/:id" do |id|
  env["cuba.vars"] ||= {}
  env["cuba.vars"]["user_id"] = id

  on photos do
    run Photos
  end

  env["cuba.vars"].delete("user_id")
end

class Photos < Cuba
  define do
    on root do
      res.write "Photos for user: %s" % env["cuba.vars"]["user_id"]
    end
  end
end
```

Of course this poses a couple of problems:

1. It becomes too dirty too quickly.
2. We have to manually manage cleanup.

Cuba::With eliminates all both issues. The following shows the
same code but rewritten to use Cuba::With:

```ruby
on "users/:id" do |id|
  with user_id: id do
    on "photos" do
      run Photos
    end
  end
end

class Photos < Cuba
  define do
    on root do
      res.write "Photos for user: %s" % vars[:user_id]
    end
  end
end
```

## Contributing to cuba-contrib.

[rum]: http://github.com/chneukirchen/rum
[rack]: http://github.com/chneukirchen/rack
[tilt]: http://github.com/rtomayko/tilt
[capybara]: http://github.com/jnicklas/capybara
[rack-test]: https://github.com/brynary/rack-test
[cuba-contrib]: https://github.com/cyx/cuba-contrib
