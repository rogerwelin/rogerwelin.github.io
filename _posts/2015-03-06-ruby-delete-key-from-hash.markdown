---
layout: post
title:  "How to delete key from a hash in ruby"
date:   2015-03-06 18:03:43
categories: ruby
---
This is the first post on this blog so I will start with something short, easy and funny - namely Ruby. Known for its elegant syntax and dynamic typing working with hashes is both easy and intuitive. Let's say you want to delete a key in a hash but you're not sure the key exists. This is one way to do it:

{% highlight ruby %}
hash = {"Miley Cyrus" => "Annoying", "Kelly Clarkson" => "Gone"}
hash.each do |k,v|
  if k.eql?("Kelly Clarkson")
    hash.delete("Kelly Clarkson")
  end
end
p hash
#=> {"Miley Cyrus"=>"Annoying"}
{% endhighlight %}

Check the [ruby doc][ruby] for more info and methods

[ruby]:        http://ruby-doc.org/core-2.2.0/Hash.html
