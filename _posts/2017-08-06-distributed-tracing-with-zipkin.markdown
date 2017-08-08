---
layout: post
comments: true
title:  "Distributed tracing with Zipkin and springcloud-sleuth"
date:   2017-08-06 11:40:43
categories: zipkin java tracing
---

## Introduction
Microservice architecture is a common architecture pattern that favors small services that's independantly deployable which communicates with each other most often with a HTTP api. 
This architecture pattern has been very popular in our industry and is heavily favoured in opposition to build big monolithic systems that are hard to develop in cooperation and deploy. However the microservice architecture pattern does come with some downsides - it's a lot harder to understand the communication model, i.e which services is comminicating with which service. And it's a lot harder and daunting to find out exactly where in the chain latency is introduced. 

If these examples above are a tad bit too abstract think about the following scenarios:

**Mapping a dependency graph for each service**  
You have a lot of microservices and you want to map exactly which services your application depends on and/or you want to know exactly which services depends on your service. In a small environment this is a no-brainer but as your architecture grows to 100+ services it's far from an easy task to map this dependency graph.

**Finding out exactly where latency is introduced in your architecture**  
Let's say you are doing a POST to _/get-customer_ and intuitively you think the call is slower than it should be. Before starting to troubleshoot a couple of question arises in regards to this perceived latency:

* When was the event and how long did it take?
* Where did this happen?
* Which event was it?
* Is it an abnormal event?
* Is the lantency a re-occuring issue or a one time thing?

Finding out the root of the latency is a hard engineering problem to solve, it's actually a lot easier to troubleshoot when operations fail than when they are just...slow.  

Turns out zipkin can help us out in both scenarios.

## What is Zipkin?
Zipkin's design is based on the [Google Dapper paper](https://research.google.com/pubs/pub36356.html) and originally developed at Twitter, Zipkin gathers timing data needed to visualize latency in microservice architectures presenting them as a waterfall graphs. So in other words zipkin allows us to compare traces to understand why certain operations took lonter time than others. 
Services need to send spans (see description in Terminology) to Zipkin for it to be able to present the information - there are a lots of libraries in different languages we can use for this but since I'll be using Java I'll be using a library called springcloud-sleuth (more on this later)

Below is a screenshot from Zipkin's website how a complete trace can look like in the web-ui:
![alt text](http://zipkin.io/public/img/web-screenshot.png "Zipkin trace")



## Terminology
* **Span** - a span is an individual operation that took place. A span contains timestamped events and tags. For example sending a RPC is a new span as is sending a response to an RPC. Span's are identified by a unique id for the span and another id for the trace the span is part of. Span's are started and stopped, and they keep track of their timing information.

* **Trace** - A trace is an end-to-end latency graphs, composed of spans. For example, if you are running a distributed big-data store, a trace might be formed by a PUT request

* **Annotation** - annotations are used to record existance of an event in time. Some of the core annotations used to define the start and stop of a request are:

**cs** : client send. The client has made a request. This annotation depicts the start of the span  
**sr** : server received. The server side got the request and will start processing it  
**ss** : server sent. Annotated upon completion of request processed (when the response got sent back to the client)  
**cr** : client received. Signifies the end of the span. The client has successfully received the response from the server side

Let's summerize: as a request arrives at a component along its journey, a new span ID is assigned for that component and added to the trace. A trace represents the whole journey of a request, and a span is each individual hop along the way, each request.

## Where does springcloud-sleuth fits in?
spring-cloud-sleuth from pivotal is a library that includes instrumentation for SpringBoot and act as streaming collector. It needs to be used by all microservices to get a complete trace of the whole request chain. It reports to Zipkin either with HTTP or other types of transports such as Kafka or RabbitMQ which is highly recommended if you plan on using it in production.
Once sprincloud-sleuth is added to the classpath it automatically instruments common communication channels (for example requests made with the springboot *RestTemplate* using HTTP headers which this example is using)


## Setting up Zipkin & start tracing
Start by cloning this repository (you will need java 8, docker and gradle installed): 

{% highlight bash %}
git clone https://github.com/rogerwelin/springcloud-sleuth-zipkin-tutorial
{% endhighlight %}

Run the following gradle command to set up zipkin as a docker container:

{% highlight bash %}
cd springcloud-sleuth-zipkin-tutorial && gradle dockerzipkin
{% endhighlight %}

You can now visit the web ui at [http://localhost:9411](http://localhost:9411), but as we haven't sent any traces to zipkin yet there isn't much to see yet. Lets fix that!  
This repo contains three different springboot applications:   

* service-lookup
* service-middleman
* service-quote

Of these it's only _service-quote_ that's actually doing something relvelant, namely a GET request on remote (internet) endpoint, parses the response and returns a string. _service-lookup_ and _service-middleman_ is dumb proxy services that chains the response from _service-quote_, but they fulfil their usage as we want a non-trivial example. The communication is then as follows:

{% highlight bash %}
service-lookup -> service-middleman -> service-quote -> remote rest endpoint (aka internet)
{% endhighlight %}

As such we start the communication chain by doing a GET request to service-lookup and if everything goes well it should produce a trace with 3 spans.


Before compiling the applications I just want to mention a couple of things:

* In build.gradle I defined the sleuth dependencies we need: *org.springframework.cloud:spring-cloud-starter-sleuth* & *org.springframework.cloud:spring-cloud-starter-zipkin*
* Every application needs to set it's name and the percentage of traces that will be captured, since this is a demo I want to capture everything (in a production scenario you only want to capture a small percentage of the traces as it will put considerable stress on zipkin). For example take a look at *applications/service-quote/src/main/resources/bootstrap.yml*


{% highlight yml %}
spring.application.name: 'service-quote'
server.port: 9000
spring.sleuth.sampler.percentage: 1.0 
{% endhighlight %}

* As for the instrumentation of the tracing we don't need to do anything else than using SpringBoots RestTemplate as mentioned before, sleuth takes care of the rest

Lets build, start the applications and do a GET on service-lookup to send trace's to zipkin:

{% highlight bash %}
✔ ~/programming/springcloud-sleuth-zipkin-tutorial [master|✔] 
15:23 $ gradle build
✔ ~/programming/springcloud-sleuth-zipkin-tutorial [master|✔] 
15:23 $ gradle printJars
:printJars
nohup java -jar applications/service-lookup/build/libs/applications/service-lookup.jar &
nohup java -jar applications/service-quote/build/libs/applications/service-quote.jar &
nohup java -jar applications/service-middleman/build/libs/applications/service-middleman.jar &
15:27 $ curl -XGET localhost:7000/lookup
Quote{type='success', value=Value{id=2, quote='With Boot you deploy everywhere you can find a JVM basically.'}}
{% endhighlight %}

Now we can head over to the zipkin ui at <localhost:9411> again and check for traces! Select *service-lookup* in the drop-down menu and click *Find Traces*. You should see something similiar to this:


<blockquote class="imgur-embed-pub" lang="en" data-id="0dhZzU2"><a href="//imgur.com/0dhZzU2"></a></blockquote><script async src="//s.imgur.com/min/embed.js" charset="utf-8"></script>

Here we get the big picture and can also drill down how the duration of each span across the services involved in the operation. You can also get more information on each span by clicking on it as shown below:

<blockquote class="imgur-embed-pub" lang="en" data-id="a/DLzgs"><a href="//imgur.com/DLzgs"></a></blockquote><script async src="//s.imgur.com/min/embed.js" charset="utf-8"></script>

Lastly we can also get the dependency graph by clicking on *Dependencies*. 

## Interactive Demo
[![asciicast](https://asciinema.org/a/jKvlLM5maVz0bDHfRqtLNkiPQ.png)](https://asciinema.org/a/jKvlLM5maVz0bDHfRqtLNkiPQ)


## Source Code
If you want to check out the complete source code you can take a look at [this repo](https://github.com/rogerwelin/springcloud-sleuth-zipkin-tutorial).

## Disclaimer
This blog post is inspired by a set of other blog post and videos on the subject and the information is borrowed heavily from them. For more information I recommend these resources:

* <https://cloud.spring.io/spring-cloud-sleuth/spring-cloud-sleuth.html>
* <https://www.youtube.com/watch?v=f9J1Av8rwCE>
* <https://www.youtube.com/watch?v=eQV71Mw1u1c>
* <http://opentracing.io>
