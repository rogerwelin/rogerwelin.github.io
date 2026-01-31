---
layout: post
comments: true
title:  "Traefik tutorial - dynamic routing for microservices"
date:   2018-09-17 19:43:43
categories: traefik reverse proxy micro services
---

Traefik is an awesome reverse proxy (and load balancer) with lots of backend supports. So why do we need another reverse proxy? (we already have nginx and haproxy that are stable and performant). Don't get me wrong, nginx and haproxy are awesome, but the problem with them is how to manage configuration. In a microservice world configuration are not stable, services come and go and configuration need to be updated to route traffic something that haproxy and nginx are not designed for. Enter traefik.

<!-- more -->

![fork join](/assets/images/traefik.svg)


Our scenario: we'll use Consul as service discovery backend to dynamically let traefik route traffic for us. 

### **Getting started**

Lets initialize the Consul backend first by starting consul from the official docker image:

```bash
$ docker run -d --name=dev-consul -p 8500:8500 consul
```

Head over to the web ui at localhost:8500 and you'll se that the service catalog is empty. Let's change that but lets set up traefik before that.

Grab the latest release as of date (I'm on mac so if you're on linux just pick linux in the github releases page):

```bash
$ wget "https://github.com/containous/traefik/releases/download/v1.6.6/traefik_darwin-amd64"
$ mv traefik_darwin-amd64 traefik && chmod +x traefik
```

Now we need to configure traefik, save the following content into a file named traefik.toml:

```bash
################################################################
# Global configuration
################################################################
debug = true
logLevel = "DEBUG"

################################################################
# Entrypoints configuration
################################################################
[entryPoints]
    [entryPoints.http]
    address = ":80"

################################################################
# API and dashboard configuration
################################################################
[api]
    entryPoint = "traefik"
    dashboard = true

################################################################
# Consul Catalog
################################################################
[consulCatalog]
endpoint = "127.0.0.1:8500"
exposedByDefault = false
prefix = "traefik"
domain = "localhost"
```

Some explanation regarding the configuration file:

* The *Entrypoint* stanza is where we specify where our proxy will route traffic from, in this case it's http://localhost:80
* In the *api* section we've just basically enabling the dashboard (located at localhost:8080)
* The *consulCatalog* is where the interesting stuff happens. Here we tell traefik to locate consul at localhost:8500, we disable *exposedByDefault*, meaning we have to explicit turn on traefik when we register a new service. The *prefix* is a string that tells traefik to care only about services that include this prefix in it's service tags.


###  **Register a service and integrate with traefik**

Start traefik with the following command:

```bash
$ sudo ./traefik -c traefik.toml
```

Now we are going to register a service in Consuls catalog, but since we are too lazy to write our own we are just going to register it with Google:s ip. Save the follwing content into a file called service.json:

```json
{
  "ID": "google",
  "Name": "google",
  "Tags": [
    "traefik.enable=true",
    "traefik.frontend.rule=Path:/api;PathStrip:/api"
  ],
  "Address": "172.217.20.36",
  "Port": 80
}
```

And do a curl against Consuls API to register the service:

```bash
$ curl -i \
> -H "Accept: application/json" \
> -H "Content-Type:application/json" \
> -XPUT -d @service.json \
> http://127.0.0.1:8500/v1/agent/service/register
HTTP/1.1 200 OK
Vary: Accept-Encoding
Date: Mon, 17 Sep 2018 22:28:00 GMT
Content-Length: 0
```

Head over to traefik's dashboard, if everything went well you should see something like this:

![fork join](/assets/images/traefik2.png)


This means that we can route traffic to our backend-google using the prefix /api. Let's give it a try:

```bash
$ curl -i -H "Host: www.google.com" localhost/api
HTTP/1.1 200 OK
...
```

It worked, great success! :racehorse:

### **Conclusion**
This post barely scratched the surface, head over to the official documentation to learn more about supported backends, tracing and configuration options; however this information should give you a rough idea how traefik works.
