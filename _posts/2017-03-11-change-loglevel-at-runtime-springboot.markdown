---
layout: post
comments: true
title:  "Springboot - change loglevel at runtime"
date:   2017-03-11 12:43:43
categories: java springboot logback
---

Both devs and ops care a lot about logging. For devs it gives them a way to peek into the behaviour of the application once it's running and way to debug, and for ops it's the first thing they go to for troubleshooting incidents in production. Because logging is such an important troubleshooting source it's really important that we can change logelvel on the fly when the default loglevel aren't showing us enough information. Unfortunaltey, from my experience, this is something we're really bad at - I can remember numerous occasions where we had to disable the node from the loadbalancer, change the loglevel, then restart the application, enable the node in the loadbalancer and repeat this dance for the x number of nodes. Turns out it's super simple to implement changing loglvel at runtime.

<!-- more -->

## Example Using Springboot and Logback
The title of this post is a tad misleading - there's really no magic springboot magic involved here. We're just accessing logbacks LoggerContext to set a new loglevel and putting a rest endpoint as a way to expose that. In the example below we're defined the method **loglevel** where we're getting the user input and the method **setLogLevel** is doing the actual work using the LoggerContext to set a new loglevel:

{% highlight java %}
    @RequestMapping(value = "/loglevel/{loglevel}", method = RequestMethod.POST)
    public String loglevel(@PathVariable("loglevel") String logLevel, @RequestParam(value="package") String packageName) throws Exception {
        log.info("Log level: " + logLevel);
        log.info("Package name: " + packageName);
        String retVal = setLogLevel(logLevel, packageName);
        return retVal;
    }

    public String setLogLevel(String logLevel, String packageName) {
        String retVal;
        LoggerContext loggerContext = (LoggerContext)LoggerFactory.getILoggerFactory();
        if (logLevel.equalsIgnoreCase("DEBUG")) {
            loggerContext.getLogger(packageName).setLevel(Level.DEBUG);
            retVal = "ok";
        } else if (logLevel.equalsIgnoreCase("INFO")) {
            loggerContext.getLogger(packageName).setLevel(Level.INFO);
            retVal = "ok";
        } else if (logLevel.equalsIgnoreCase("TRACE")) {
            loggerContext.getLogger(packageName).setLevel(Level.TRACE);
            retVal = "ok";
        } else {
            log.error("Not a known loglevel: " + logLevel);
            retVal = "Error, not a known loglevel: " + logLevel;
        }
        return retVal;
    }
{% endhighlight %}

After starting the server you can issue a simple http POST to the rest endpoint we defined above and choose the loglevel you want to change to and the package you want to it affect:

{% highlight bash %}
curl -XPOST localhost:8080/loglevel/debug?package=se.roger
{% endhighlight %}


## Demo
[![asciicast](https://asciinema.org/a/60g6smj2c330fvg4j0a848zmd.png)](https://asciinema.org/a/60g6smj2c330fvg4j0a848zmd)


## Complete Example
If you want to try out a complete example you can check out [this repo](https://github.com/rogerwelin/springboot-change-loglevel-runtime) and follow the README.
