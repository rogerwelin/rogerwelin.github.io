---
layout: post
comments: true
title:  "Convert markdown to confluence markup"
date:   2017-05-15 11:43:43
categories: markdown confluence docker
---

I recently did some investigations at work on how to keep documentation up to date. I guess we've all been in the situation that the documentation we have is drifting from how the actual systems/applications look like. I believe that it's easier to keep the documentation close the the actual code, e.g by README's in markdown, than by have them as separate confluence pages that no one remembers to keep up to date. If you are using Confluence you can create/update documentation by using the REST api, however Confluence does not accept markdown so you have two options; 

<!-- more -->

* install markdown plugins (that's available in the api)
* or convert markdown to confluence markup language 

I went with the latter options since that felt a bit easier. I found this [ruby gem](https://github.com/jedi4ever/markdown2confluence). Unfortunately it's installed as a gem with dependencies you might not have available and it only runs as a cli-tool which might not be ideal if you want to include this as a step in your CI/CD pipeline. I quickly hacked together a way to run this gem as a REST server instead inside a docker container, which eliminates bothersome dependencies and make's it easier to include in your pipeline.

## Quick Tutorial

* Run the container by pulling the image from docker hub:

```bash
docker run -d -p 9292:9292 rogerw/markdown2confluence-server
```

* Test the conversion (example script in ruby)

```ruby
#!/usr/bin/ruby
require 'json'
require 'httparty'

file = File.open('/path/to/README.md').read
endpoint = "http://localhost:9292/markdown2confluence"

markdown_hash = Hash.new
markdown_hash[:content] = file

response = HTTParty.put(endpoint, :body => markdown_hash.to_json)
puts response.body

# code to push to confluence here
# example: https://developer.atlassian.com/confdev/confluence-server-rest-api/confluence-rest-api-examples
```


## Asciicast Demo
[![asciicast](https://asciinema.org/a/54i3clbggpayy28d0weeef572.png)](https://asciinema.org/a/54i3clbggpayy28d0weeef572)


## Complete Example
If you want to check out the complete source code you can take a look at [this repo](https://github.com/rogerwelin/markdown2confluence-server) and follow the README.
