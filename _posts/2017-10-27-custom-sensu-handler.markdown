---
layout: post
comments: true
title:  "Write and configure custom sensu handlers"
date:   2017-10-27 11:43:43
categories: markdown confluence docker
---

## Introduction
Sensu is a decent monitoring platform comparing to nagios, since the configuration is more flexible (more config mgmt friendly) and it's easier to scale. In this post I will demonstrate how easy it is to configure cusom handlers on checks and how to send notifications.

## Setting it up
What I'mn trying to accomplish is the following:
- checks are run standalone
- all checks should run a custom handler
- notifications should be sent to my prefered chat-tool
- notification should only be sent on state changes (eg. from OK to Critical or from Critical to OK), to avoid getting spammed.

Lets look at the configuration needed first. Handler config is only needed on sensu-server, lets add the following config to /etc/sensu/conf.d/slack-handler.json

{% highlight json %}
{
  "handlers": {
    "slack-handler": {
      "type": "pipe",
      "filter": "state-change-only",
      "command": "/etc/sensu/plugins/slack-handler.rb"
    }
  }
}
{% endhighlight %}

A pipe handler means basically that after a check is run we will also run a external command, the command will get data feed into it from STDIN (check information etc.). **Filter** is something we can apply to say when to run the handler, in this case we only want to run the handler on state changes so let's look at how to accomplish that; on the sensu-server add the following file: /etc/sensu/cond.f/state-change.only.json

 
{% highlight json %}
{
  "filters": {
    "state-change-only": {
      "negate": false,
      "attributes": {
        "occurrences": "eval: value == 1 || ':::action:::' == 'resolve'"
      }
    }
  }
}
{% endhighlight %}

I will not go into detail about this configuration besides that it enables us to only run the handler on state-changes. For more information read the sensu docs. Lets write the handler now! Add the following file /etc/sensu/plugins/slack-handler.rb:


{% highlight ruby %}
#!/usr/bin/env ruby
require 'json'
require 'net/http'
require 'net/https'
require 'uri'


url_hook = 'http://url-webhook-token'

# Read the incoming JSON data from STDIN.
event = JSON.parse(STDIN.read, :symbolize_names => true)

check_name   = event[:check][:name]
check_output = event[:check][:output]
check_fqdn   = event[:client][:name]
check_status = event[:check][:status]

text = String.new


table="
| Host               | Check           | Output                    |
| :----------------- |:--------------- | :-------------------------|
| #{check_fqdn}      | #{check_name}   | #{check_output}           |
"

if check_status == 0
  text =  "### :white_check_mark: The following sensu check has been resolved\n#{table}"
else 
  text = "### :warning: Alert from sensu\n<!channel> please review the following alert.\n#{table}"
end


alert_hash = {:username => 'sensu', :text => text}


uri = URI(url)
http = Net::HTTP.new(uri.host, uri.port)
http.read_timeout = 10
http.use_ssl = true
http.verify_mode = OpenSSL::SSL::VERIFY_NONE
req = Net::HTTP::Post.new(uri.path, 'Content-Type' => 'application/json')
req.body = alert_hash.to_json
res = http.request(req)
puts "response #{res.body}"
{% endhighlight %}


Great! So the handler script gets data from sensu from STDIN which we store in the event hash. From it we can get data such as the name of the chack, the text output and the status. Depending on the status (Critical or OK) we send a markown-table to the slack channel with different messages. Now the only thing that is left is to configure each standalone check we have to use this handler, this is configured on the sensu-clients. An example could be:


{% highlight json %}
{
  "checks": {
    "check-process": {
      "standalone": true,
      "interval": 60,
      "command": "/etc/sensu/checks/check-process.sh -p [name]",
      "subscribers": [ "ALL" ],
      "handlers": [ "slack-handler" ]
    }
  }
}

{% endhighlight %}

The only thing for importance here is the handlers value which is "slack-handler" which we defined earlier on the sensu-server. 
