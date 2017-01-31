---
layout: post
title: composing http endpoints
categories: apis
---

While I've been working on
[choclety](https://github.com/mooreniemi/choclety), I've also been
experimenting a bit. Choclety is just based on the observation that many
times a return type of one endpoint (some representation of a resource) is
an input to another. Relatedly I thought, wouldn't it be fun to try
composing endpoints, ie.


```
GET http://localhost:4567/a
# => {"a":"a"}
GET http://localhost:4567/a?compose=!b
# => {"b":"a + b"}
GET http://localhost:4567/a?compose=!b,c
# => {"c":"a + b + c"}
```

Here's a working implementation in Ruby, using [Sinatra](www.sinatrarb.com):

```ruby
# prereqs:
# gem install sinatra
# gem install sinatra-contrib

require 'sinatra'
require 'json'
require 'sinatra/json'
require 'addressable/uri'

COMPOSITION_PREFIX = /([!\-+]?)(.*)/
COMPOSITION = {'+': 'PUT', '!': 'POST', '-': 'DELETE'}
COMPOSITION.default = 'GET'
COMPOSITION.freeze

helpers do
  def compose(params, response, path)
    prefix, path = COMPOSITION_PREFIX.match(path)[1], COMPOSITION_PREFIX.match(path)[2]

    query_string = Addressable::URI.new(
      query_values: response.merge(params)
    ).query

    env_hash = {
      'PATH_INFO' => "/#{path}",
      'QUERY_STRING' => query_string,
      'BODY' => response,
      'REQUEST_METHOD' => COMPOSITION[prefix.to_sym]
    }

    p "composing #{path}"
    call env.merge(env_hash)
  end

  def respond(response)
    composition_stack = (params.delete('compose') || "").split(',')
    if t = composition_stack.shift
      params.merge!({ compose: composition_stack.join(',') })
      status, headers, body = compose(params, response, t)
      [status, headers, body]
    else
      json(response)
    end
  end
end

get '/a' do
  p 'responding from a'
  respond({a: "a"})
end

post '/b' do
  a = params[:a]
  p 'responding from b'
  respond({b: "#{a} + b"})
end

get '/c' do
  body = env['BODY']
  p 'responding from c'
  respond({c: ((body || {}).values + ['c']).join(' + ')})
end
```

The theoretical advantage to this is that you could collapse many requests
in a single one. It's dangerous though! In my example, you can compose via
a GET and essentially force a POST to happen.

What's more valuable about an example like this, to me, is simply that it
surfaces an interesting constraint. If you force an API to be consistent
across input types and return types, you can chain or compose calls
provided you don't need to parameterize the calls further.

You generally will need to parameterize API calls with user interaction.
This emphasizes that integration point with your user, but also means you
could write 'adapters' for other agents.
