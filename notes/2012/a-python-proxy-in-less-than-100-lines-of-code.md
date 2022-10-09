---
[meta]
date = 2012-08-29
description = "Learn how to write a Python proxy using raw sockets and the python standard library."
tags = "python, proxy, socket, tutorial, howto, network"
title = "A python proxy in less than 100 lines of code"
---
##What is a tcp proxy?##

It's a intermediary server intended to act in name of a client, and sometimes to do something useful with the data before it reaches the original target. Let's see a picture:

![Proxy](https://upload.wikimedia.org/wikipedia/commons/thumb/b/bb/Proxy_concept_en.svg/277px-Proxy_concept_en.svg.png)

My idea was to produce a proxy using only the default python library, and to guide me during development I set the following:

* Each new client connection to our proxy must generate a new connection to the original target. 
* Each data packet that reaches our proxy must be forwarded to the original target.
* Each data packet received from the target must be sent back to the correct client
* The proxy must accept multiple clients
* Must be fast	
* Must have a low resource usage

##The result:##

<script src="https://gist.github.com/voorloopnul/415cb75a3e4f766dc590.js"></script>

##The explanation##

###class Forward()###

The Forward class is the one responsible for establishing a connection between the proxy and the remote server(original target).

###class TheServer().main_loop()###

The input_list stores all the avaiable sockets that will be managed by select.select, the first one to be appended is the server socket itself, each new connection to this socket will trigger the on_accept() method.

If the current socket *inputready* (returned by select) is not a new connection, it will be considered as incoming data(maybe from server, maybe from client), if the data lenght is 0 it's a close request, otherwise the packet should be forwarded to the correct endpoint.

###class TheServer().on_accept()###

This method creates a new connection with the original target **(proxy -> remote server)**, and accepts the current client connection **(client->proxy)**. Both sockets are stored in input_list, to be then handled by main_loop. A "channel" dictionary is used to associate the endpoints(client<=>server). 

###class TheServer().recv()###

This method is used to process and forward the data to the original destination ( client <- proxy -> server ).

###class TheServer().on_close()###

Disables and removes the socket connection between the proxy and the original server and the one between the client and the proxy itself. 

That's it.
