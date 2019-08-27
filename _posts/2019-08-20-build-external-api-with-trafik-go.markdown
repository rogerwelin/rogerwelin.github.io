---
layout: post
comments: true
title:  "Build an external api with auth using Traefik and Go"
date:   2019-08-19 18:43:43
publish: true
categories: traefik api go auth
---

In this post I will show how to easily build an external api with authentication with help from Traefik. The usecase is the following: you want to make an external api only authenticated users can access. In this solution the authentication will be a token which clients will supply as an HTTP header. We will leverage Traefik to make the auth server and underlying api services loosely coupled.


Our flow will look like this: all request to out api service will firstly be forwarded to an auth server, if the auth server responds with 200, access is granted and the clients original request will be performed. Otherwise the request from the auth server will be performed which will be an 401. We will configure Traefik to handle this routing for us. Take a look at the picture below for the flow of the request:

![fork join](/assets/images/authforward.png)

<!-- more -->

### **Setting up Traefik and Consul**

We will use Consul as a discovery backend for Traefik where we will register our auth and api service. I've already written a basic tutorial on how to set those up, please read the *Getting started* section [here](https://rogerwelin.github.io/traefik/reverse/proxy/micro/services/2018/09/17/traefik-tutorial.html) on how to set them up before continuing in this guide.

###  **The API and auth server**

To keep things easy we will combine our api service and our auth server into one app. Take a look at the code below. The code is fairly self-describing but important sections are commented.


{% gist 7dc9bcf7e2d4a92ad1a3fcb367df41ac %}


### **Assemble all the pieces**

With the api/auth server done, we can start firing up Traefik, Consul and our api/auth server:

````bash
$ docker run -d -p 8500:8500 consul:1.5.1
$ ./traefik -c traefik.toml &
$ go run server.go
```

Now we are going to register our auth/api service to Consul, save the json blob below into a file called service.json:

{% gist 71bb4416ff06ad1f5accf6f8f3edaf86 %}

*frontend.rule=Path* tells Traefik that we want to claim the /api endpoint. And with *frontend.auth.forward.address* tells Traefik to forward all request first to our /auth endpoint before continuing with the original request. *frontend.auth.forward.authResponseHeaders* tells Traefik which headers to copy from the auth server.


Do a curl against Consuls API to register the service so Traefik will pick it up:

````bash
$ curl -i \
> -H "Accept: application/json" \
> -H "Content-Type:application/json" \
> -XPUT -d @service.json \
> http://127.0.0.1:8500/v1/agent/service/register
HTTP/1.1 200 OK
Vary: Accept-Encoding
Content-Length: 0
```

Let's try our solution with a couple of curl requests (also take a look at the logs on stdout of the api/auth server):

````bash
$ curl -i  localhost/api
HTTP/1.1 401 Unauthorized

$ curl -i -H "X-Auth-Token: wrong-token" localhost/api
HTTP/1.1 401 Unauthorized

$ curl -i -H "X-Auth-Token: abc123" localhost/api
HTTP/1.1 200 OK

you're authenticated acmecorp
```

### **Conclusion**
We've implemented an api with authentication that is ready for external clients in under 50 loc, that's pretty cool! This is of course a toy example but I really like how Traefik can make the different parts loosely coupled.


