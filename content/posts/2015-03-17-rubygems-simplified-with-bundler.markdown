---
layout: post
comments: true
title:  "Rubygems made easy with bundler"
date:   2015-03-17 20:03:43
categories: ruby gems bundler
---

One thing with Ruby I don't quite like (and many with me) are the Ruby gems. The one thing thats great with gems is that it's almost as with CPAN, if you have a problem - there's a gem for that! But I don't like the idea of installing libraries system-wide. If installed with sudo it's a security risk. And my belief is that libraries are supposed to be used by demand when an application needs them. For example Java + maven and Node.js and NPM is a good example of those getting this right. Let's say you want to share a ruby project, the problem is  that person must have the same set of gems installed to be able to run your program, and maybe he's installing a newer version of that gem that you was testing on. Not good! Fortunately we have Bundler to help us keep track of gems and it's dependencies that solves this problem.

<!-- more -->

Let's create a simple ruby project where we want install a set of gems locally to that application and how we can share this project to another machine with ease. First of Bundler is a gem so we need to get that:

```
gem install bundler
```

Then type ```bundle init``` to set up an empty Gemfile which we will populate with gems:

```ruby
source 'https://rubygems.org'
gem 'httparty', '0.3.13'
gem 'json'

group :development do
  gem 'dm-sqlite-adapter'
end

group :production do
  gem 'pg'
  gem 'dm-postgres-adapter'
end
```


The first line tells Bundler where to get all your gems. Here we're using rubygems.org but you could have other in-house hosted repositories that you can use instead. Rubygems.org should be a sane default though. After that we are listing all the gems we want to use in our simple app. Note that we have pinned the version of httparty to 0.3.13, this ensures us that we are always using the same version. 

Note that we have grouped certain gems in a group block; development and production. Grouping your gems allows us to perform operation on the entire group, which I will show later on. In this example we are using sqlite in development will in production we want to use a postgres driver.

Not let's make Bundler do some heavy lifting for us and installing some gems:

```sh
$ bundle install --without production --path vendor/bundle

Fetching gem metadata from https://rubygems.org/.........
Fetching version metadata from https://rubygems.org/..
Resolving dependencies...
Installing addressable 2.3.7
Installing data_objects 0.10.15
Installing dm-core 1.2.1
Installing dm-do-adapter 1.2.0
Installing do_sqlite3 0.10.15
Installing dm-sqlite-adapter 1.2.0
Installing json 1.8.2
Installing multi_xml 0.5.5
Installing httparty 0.13.3
Using bundler 1.8.5
Bundle complete! 5 Gemfile dependencies, 10 gems now installed.
Gems in the group production were not installed.
Bundled gems are installed into ./vendor/bundle.
```


Three important things here:

* ```--without production``` flag enables us to install all the default gems (those without a group declaration) and all other gems that are not in the production group

* With the ```--path``` flag we tell bundler that we want the gems installed in the project root folder under vendor/bundle

* You'll notice that bundler has written a file named ```Gemfile.lock```, this is an important file. From bundlers doc:

>After developing your application for a while, check in the application together with the Gemfile and Gemfile.lock snapshot. Now, your repository has a record of the exact versions of all of the gems that you used the last time you know for sure that the application worked. 
This is important: the Gemfile.lock makes your application a single package of both your own code and the third-party code it ran the last time you know for sure that everything worked. Specifying exact versions of the third-party code you depend on in your Gemfile would not provide the same guarantee, because gems usually declare a range of versions for their dependencies.


Now it's time to run our app. But first we need to make sure we tell our program to use the gems that bundler provided us with and not Rubys system gem path. You only need to require bundler/setup for this as shown below:


```ruby
#!/usr/bin/ruby
require 'bundler/setup'
require 'httparty'

10.times do 
  #psuedo code
end
```
