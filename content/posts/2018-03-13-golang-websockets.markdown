---
layout: post
comments: true
title:  "Using websockets in golang with gorilla-websocket"
date:   2018-03-13 19:43:43
categories: golang websockets gorilla
---

This will be a tutorial in how to implement websockets in Go using the excellent gorilla/websocket package.
We will build a backend server in Go that accept json from the user and pushes that data on the websocket channel; the frontend will be a simple html file with inline javascript (for simplicity sake) that connects to the go backend and listens on data and updates the html in realtime.

<!-- more -->

## Brief Description of the Websocket Protocol
The quick explanation on wikipedia on websocket goes like this:

> WebSocket is a computer communications protocol, providing full-duplex communication channels over a single TCP connection

Duplex meaning two-way communication on the same connection. Websocket still operates on TCP, though the protocol provides multiple benefits over the regular http protocol. As the server and client establish a persistent connection, we have an ideal way of pushing data to the client without having to resort to polling. It also brings significant performance benefits as we don't have the overhead of opening multiple TCP connections and we can push as many data through the connection as we want without the overhead of traditional HTTP requests.

The establishment of a new websocket connection in a simplified form goes like this:
* The client initiates the connection sending a regular HTTP request to the server (using the **ws://** scheme). An Upgrade heade is included in the request making the server aware that it will be a ws connection
* The server agrees (if it supports websockets) and communicates this through an Upgrade header in the response. Now the handshake is complete and the initial HTTP connection is replaced by a websocket connection

## Fetch Dependencies
We'll grab both gorilla/mux (although mux is not strictly needed I use if for convenience) and gorilla/websockets:

* *go get -u github.com/gorilla/mux*
* *go get -u github.com/gorilla/websockets*

## Let's Start Coding
We will start with building the server. The server will set up one endpoint for the user to POST data to and one endpoint for the websocket connection. We will send a POST with curl with longitude and latitude data that will propagate through the websocket endpoint to the client.
Take note of the provided comments in the code, I'll step through them in order:

1. This is our data struct, we'll Unmarshal this after the user has sent the json
2. Our main func, here we setup 3 routes using gorilla/mux; handler **longLatHandler** will unmarshal the json and write the struct to a channel, handler **wsHandler** is responsible for negotiationg the websocket connection with the client
3. After wsHandler initiated the websocket connection, we step into the echo func where we will loop infinitely. In this loop we are reading from the queue channel (which will be populated by http POSTs), when we have a message on the queue we are sending that on the conn using the WriteMessage func as a slice of bytes


{{< gist rogerwelin 4d9891b2440e62e88ce34f9f7bdc31b7 >}}

Now lets take a look at the client html/js (as with the go code I'll discuss the comments in numerical order):

1. Two div's (one for lat and one for long) which the javascript will update
2. The variable *exampleSocket* initiates the websocket connection to the server
3. On a websocket event we split the message into an array and write them to the div's

{{< gist rogerwelin 6b09cfa5c485428a4be0b36ed8da4c4c >}}

## Demo
Let's put everything together now! First start the websocket server:

```sh
$ go run websocket.go
```

Save the html/js in a file and open it in Chrome or Firefox. It will only show a header initially, lets change that; curl our websocket server with the following curl:

```sh
$ curl -H "Accept: application/json" -XPOST -d '{"longitude": 18.063240, "latitude": 59.334591}' localhost:8844/longlat
```

Head over to the browser and see the result. Try a couple of other curl's and see how we successfully push data to the client without needing to reaload the page :octocat:
