---
layout: post
comments: true
title:  "Creating helpful command line applications in ruby"
date:   2016-01-14 20:43:43
categories: ruby
---

## Introduction

So what makes a good and helpful command line application? I think these three point are important:

* easy to use
* helpful
* play well with others

Bash is commonly used to write command line apps, but I can never shrug of the feeling of shame of writing a bash script with more than 30 lines.
That's why I'm shaking things up here and showing how to use ruby instead. I'll be using the optparse gem so you need to do a ```gem install optparse```
before trying out the shown examples.

# A bad example

This is how a bad command line application looks like (it will try to take a backup of a mysql database):

{% highlight ruby %}
#!/usr/bin/env ruby

# backup mysql database

db          = ARGV.shift
username    = ARGV.shift
password    = ARGV.shift
date        = Time.now.strftime('%Y%m%d')
backup_file = "/tmp/backup_#{date}.sql"
mysqldump   = "mysqldump -u #{username} -p #{password} #{database}"

system("#{mysqldump} > #{backup_file}")
system("gzip #{backup_file}")
{% endhighlight %}

This is bad for a various of reasons. First of all the user will have no idea how to run this script without looking at the source code. Second, the script doesn't have input validation, it will gladly just take one parameter, even though 4 parameter is required. Third, using the system command will not give the user any feedback.


# A better example with optparse

First thing we'll create an empty hash that will store all our options. In the OptionParser block we'll assign both flags and switches making the user able to execute the utility as they choose. The opt.banner will be printed when no arguments was passed to the program or wrong number of argument, making the program user-friendly.

{% highlight ruby %}
#!/usr/bin/ruby
require 'optparse'
require 'logger'

logger = Logger.new(STDOUT)
current_file = File.basename(__FILE__)

# if options undefined set to help option
if ARGV.empty?
  ARGV[0] = '-h'
end

options = {}

opt_parser = OptionParser.new do |opt|
  opt.banner = "Command-line utility backing up mysql databases
  ./#{current_file} [-d name of database] [-u username] [-p password]
  Example: ./#{current_file} -d one_punch_man -u user -p password
"

  opt.on("-d","--database [Database]","The name of the database") do |db|
    options[:db] = db
  end

  opt.on("-u","--username [Username]","The database username") do |user|
    options[:user] = user
  end

  opt.on("-p","--password [Password]","The database password") do |password|
    options[:password] = password
  end

  opt.on("-h","--help","help") do
    puts opt_parser
    exit 1
  end
end

opt_parser.parse!
database = options[:db]
username = options[:user]
password = options[:password]

logger.info("Command-line options accepted")
logger.info("Start backup")
{% endhighlight %}


Running the program with no arguments will look like this:

{% highlight sh %}
Command-line utility backing up mysql databases
  ./mysql.rb [-d name of database] [-u username] [-p password]
  Example: ./mysql.rb -d one_punch_man -u user -p password
    -d, --database [Database]        The name of the database
    -u, --username [Username]        The database username
    -p, --password [Password]        The datbase password
    -h, --help                       help
{% endhighlight %}

And running the program with the correct arguments:

{% highlight sh %}
nordnet@NNs-MacBook-Pro:~/Programming/ruby$ ruby mysql.rb -d one_punch -u user -p pass
I, [2016-01-14T20:15:24.368245 #10280]  INFO -- : Command-line options accepted
I, [2016-01-14T20:15:24.368322 #10280]  INFO -- : Start backup
{% endhighlight %}

Optparse rocks!

