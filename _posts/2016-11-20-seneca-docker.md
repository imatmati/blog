---
layout: post
title: "Microservices with Seneca and Docker"
description: "Microservices are the new trend in composing applications with small and reusable pieces of logic accessible from any clients.But if services should be light and tiny as possible, what about their execution environment ? Can I expect to run hundreds of microservices each with their own and heavy application server ? If so, application server would demand much more resources than our little service, the ratio would be catastrophic. Is there a solution to run microservices on micro application servers and why not on micro containers ? Yes, the solution is Seneca plus Docker."
tags: [Microservice,Seneca, Docker]
---

Microservices are the new trend in composing applications with small and reusable pieces of logic accessible from any clients.But if services should be light and tiny as possible, what about their execution environment ? Can I expect to run hundreds of microservices each with their own and heavy application server ? If so, application server would demand much more resources than our little service, the ratio would be catastrophic. Is there a solution to run microservices on micro application servers and why not on micro os ? Yes, the solution is Seneca plus Docker.

Think of it, why would you use microservices ?
Here a few common reasons :
- Composable services
- Horizontal scalability
- Logic isolation.
- API management.
- Heterogeneous clients.

One of them is the most underestimated and the most technical : scalability.
But how do you currently implement your microservice ? Did you really take scalability into consideration ? Or more probably you stuck to your good old technologies so often successful in the past.

Let me present to you what may be your next try to address scalability in your development.

# Seneca 

> Seneca is a microservices toolkit for Node.js. 
> It helps you write clean, organized code that you can scale and deploy at any time.

I took this presentation literally from their site, it's a perfect fit. Maybe I'd add service discovery as a major feature worth being mentioned.

Now, how would I code my first and simple microservice ?
First, install Node.js if necessary and create your project in a directory of your choice by :

{% highlight shell %}
$ npm init
{% endhighlight %}

Then install Seneca by :
{% highlight shell %}
$ npm install seneca --save
{% endhighlight %}

To test our first service, get some fake users from [https://www.mockaroo.com/](https://www.mockaroo.com/) and save them in json format in your project directory.

Now comes our first microservice with seneca that serves users data. Copy the code in index.js

{% highlight javascript %}
var seneca = require ("seneca")();
var data = require ("./MOCK_DATA.json");

seneca.add("role:users,cmd:get", function(msg, respond) {

	var user = data.filter(function(usr) { return usr.id == msg.id});
	var error =null;
	if (user.length === 0 ) {
		error = Error ("user not found");

	}
	else{
		user = user[0]
	}
	
	respond(error, {"user" : user})
})

seneca.listen();
{% endhighlight %}

What happens here ?
First we create our seneca instance. It's versatile and can serve to implement server and client as well.
We'll use it as a server to pass user data from our file of users.

Seneca uses pattern matching to identify a service. And we litteraly add a service with ... add function.
First parameter is the pattern identifier of this service. This pattern matching is wonderful to classify our microservices and permits some form of overloading with a common part in identifiers and a distinguible one to route correctly to the target. 


Then the service logic is provided as a function.
This function accepts a message argument with all arguments passed to the call of this web service.
Respond is our "classical" callback with its first parameter to signal error and the second one to send data back to the client.

Finally, we launch our service with "listen". Seneca is able to listen with different transports protocols and ports.
The default transport plugin provides HTTP and TCP protocols.
But for all the examples, we'll keep HTTP.
By default, seneca server listens on port 10101 with HTTP protocol.

What if you want to change the listening port number ?  Simple, just pass the option to listen function

{% highlight javascript %}
seneca.listen({port:"5000"});
{% endhighlight %}

And if I want to listen from a specific network address ? Same way. Here a dummy example with localhost.

{% highlight javascript %}
seneca.listen({host:"localhost",port:"5000"});
{% endhighlight %}

Ok, how can I call it ? 
Two ways are offered, you can use Seneca client or any HTTP client.

### HTTP client

{% highlight shell %}
$ curl -d '{"role":"users","cmd":"get","id":4}' http://127.0.0.1:5000/act
{"user":[{"id":4,"first_name":"Anna","last_name":"Stephens","email":"astephens3@deviantart.com","gender":"Female","ip_address":"194.18.255.45"}]}
{% endhighlight %}

Don't forget the act path in your URL, it's the marker of any client calls even from Seneca ones.

### Seneca client
To demonstrate Seneca client, just execute the following codes respectively inside and outside index.js.

* Inside the same file index.js
{% highlight javascript %}
seneca.act("role:users,cmd:get,id:4", function (err,response) {
	if (err) return console.log (err.msg);
	console.log (response);
})
{% endhighlight %}

* Outside in client.js we must specify the targeted service in the client call.
{% highlight javascript %}
var seneca = require ("seneca")();
seneca.client({host:"127.0.0.1", port:5000}).act({"role":"users","cmd":"get","id":4}, function (err,response) {
	if (err) return console.log (err.msg);
	console.log (response);
})
{% endhighlight %}

As seen above, you specify the pattern identifier of the service you want to call along with the data to transmit.
An other way to write this part would be in JSON format.


{% highlight javascript %}
seneca.act({"role":"users","cmd":"get","id":4}, function (err,response) {
	if (err) return console.log (err.msg);
	console.log (response);
})
{% endhighlight %}

Even if we're not going to use it in our examples, note that you can export your service as an Express middleware to use it in your existing Express application.
 
### Service discovery



Seneca provides a plugin for service discovery called 'mesh'. And there shines the concept of pattern. Remember a service is identified by the pattern it takes in charge. Mesh permits to services to address each other thanks to this pattern regardless of current location of the endpoint.

First, install the plugin in your project.

{% highlight shell %}
$ npm install seneca-balance-client
$ npm install seneca-mesh
{% endhighlight %}

Then our service discovery needs a node to be the entry point to join the network, it's called the base. Notice that it will be used only once by a client joining the network. In any subsequent need for information about service location any node can answer the request. That means that if the base goes down, our cluster will still work with its actual members !

I chose to implement a base within an actual service.Of course, you can decide to externalize a devoted instance for that matter.
If necessary you can simply create a base with a single line of code
{% highlight shell %} 
$ node -e 'require("seneca")().use("mesh",{isbase:true})'
{% endhighlight %}

This is the code for the intermixed variation.

{% highlight javascript %}
var seneca = require ("seneca")();
var data = require ("./MOCK_DATA.json");

seneca.add("role:users,cmd:get", function(msg, respond) {

	var user = data.filter(function(usr) { return usr.id == msg.id});
	var error =null;
	if (user.length === 0 ) {
		error = Error ("user not found");
	}
	else{
		user = user[0]
	}
		
	respond(error, {"user" : user})
}).use('mesh',{isbase:true,"pin":"role:users,cmd:get"})
{% endhighlight %}

New things happen here. We use the plugin 'mesh' and make our service a base for our network of services with isbase:true option.
We notify the (future) members that we publish a service identified by "role:users,cmd:get". Pay attention to the fact, we don't listen to a port.
So our service is only discoverable by others members of the network but not directly addressable.
If we aim to be able to address it in a direct call we should have built a base outside and make a listen call as follow

{% highlight javascript %} 
seneca.add("role:users,cmd:get", function(msg, respond) {

	var user = data.filter(function(usr) { return usr.id == msg.id});
	var error =null;
	if (user.length === 0 ) {
		error = Error ("user not found");
	}
	else{
		user = user[0]
	}
	
	respond(error, {"user" : user})
}).use('mesh',{"pin":"role:users,cmd:get"}).listen({port:6000,"pin":"role:users,cmd:get"})
{% endhighlight %}

Why do I double the pin option ? It'll become clear in a moment.

Now, the client service

{% highlight javascript %} 
var seneca = require ("seneca")();

seneca.add("role:users,cmd:check", function (msg, respond) {

	//First get the user
	seneca.act({"role":"users","cmd":"get", "id":msg.id}, function (err, response) {

		if (err) return console.log (err)

			// Do check
			// ........
			response.user.checked=true;
			respond(null,response)
		})

}).use('mesh',{pin:"role:users,cmd:check"}).listen({port : 5000,pin:"role:users,cmd:check"})
{% endhighlight %}

The service identified by "role:users,cmd:check" will make a call to an other service identified by "role":"users","cmd":"get".
Note that no address is given here, only the identifier. We indicate our pattern to identify the service as on option to the mesh plugin.
To make it callable, we listen on port 5000.

Launch the two services with node and issue a call by

{% highlight shell %} 
$ curl -d '{"role":"users","cmd":"check","id":3}' http://127.0.0.1:5000/act
{% endhighlight %}
The answer
{% highlight shell %} 
{"user":{"id":3,"first_name":"Ralph","last_name":"Riley","email":"rriley2@cdbaby.com","gender":"Male","ip_address":"239.75.131.248","checked":true}}
{% endhighlight %}

Of course the response will differ according to you generated test data.

How do we know what services are exposed in a network ?
Fairly simple, call this on any service of the network.

{% highlight shell %} 
$ curl -d '{"role":"mesh","get":"members"}' http://127.0.0.1:6000/act
{% endhighlight %}

the answer contains all the services 

{% highlight shell %} 
{"list":[
{"pin":"role:users,cmd:get","port":6000,"host":"0.0.0.0","type":"web","instance":"99ni7imqrbn1/1479846054979/7136/3.2.2/-"},
{"pin":"role:mesh,base:true","port":52631,"host":"0.0.0.0","type":"web","model":"actor","instance":"hm32xs56zucx/1479845435857/6642/3.2.2/-"},
{"pin":"role:users,cmd:check","port":5000,"host":"0.0.0.0","type":"web","instance":"682daw0mery3/1479846059980/7141/3.2.2/-"},
{"pin":"role:users,cmd:check","port":59982,"host":"0.0.0.0","type":"web","model":"actor","instance":"682daw0mery3/1479846059980/7141/3.2.2/-"},
{"pin":"role:users,cmd:get","port":6000,"host":"0.0.0

.0","type":"web","instance":"99ni7imqrbn1/1479846054979/7136/3.2.2/-"},
{"pin":"role:mesh,base:true","port":52631,"host":"0.0.0.0","type":"web","model":"actor","instance":"hm32xs56zucx/1479845435857/6642/3.2.2/-"},
{"pin":"role:users,cmd:check","port":5000,"host":"0.0.0.0","type":"web","instance":"682daw0mery3/1479846059980/7141/3.2.2/-"},
{"pin":"role:users,cmd:check","port":59982,"host":"0.0.0.0","type":"web","model":"actor","instance":"682daw0mery3/1479846059980/7141/3.2.2/-"}]}
{% endhighlight %}

Here appears the reason to double the pin of services : to make it appear in the network service as well as in the direct callable port.
For example, you obtain "pin":"role:users,cmd:check","port":5000 for the "callable" part and "role:users,cmd:check","port":59982 for the "hidden" but known service in our network. If you delete any one the pin option, you only receive a not very expressive "pin":"null:true". 
And you've got the same information twice because two nodes are in our network and each one provides its information.

# Docker

I won't waste to present Docker. What I try here is to show you an introduction to how a docker microservices network could be built.

### Get user

The first service will serve as a base and be attributed to its own container.
This is its Dockerfile.

{% highlight shell %} 
FROM ubuntu:14.04
MAINTAINER ivan matmati
EXPOSE 39999
ENV PATH /opt/node/bin:$PATH
ADD node /opt/node
WORKDIR /logi/getuserservice
RUN apt-get update
RUN apt-get -y install python
RUN apt-get -y install gcc
RUN apt-get -y install make
RUN apt-get -y install g++
ADD package.json index.js MOCK_DATA.json ./
RUN /opt/node/bin/npm install
CMD /opt/node/bin/node index.js 
{% endhighlight %}

I choose ubuntu 14.04 only because it's my current system. For the base to be able to communicate with exterior, I expose the default port 39999.
To use the latest version of node, I had to provide it myself as a directory in the project directory and copy it into /opt/node. I also created a special directory for my service /logi/getuserservice and copy all necessary artefacts : package.json index.js and MOCK_DATA.json

The version of code I used for the service in index.js is 
{% highlight javascript %} 
var seneca = require ("seneca")();

var data = require ("./MOCK_DATA.json");
seneca.add("role:users,cmd:get", function(msg, respond) {

	var user = data.filter(function(usr) { return usr.id == msg.id});
	var error =null;
	if (user.length === 0 ) {
		error = Error ("user not found");
	}
	else{
		user = user[0]
	}
		
	respond(error, {"user" : user})
}).use('mesh',{isbase:true,"pin":"role:users,cmd:get"})
{% endhighlight %}

My package.json is dedicated to seneca

{% highlight json %}
{
  "name": "ws1",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "seneca": "^3.2.2",
    "seneca-balance-client": "^0.6.0",
    "seneca-mesh": "^0.9.0"
  }
}
{% endhighlight %}

Let's build it. My project for this sole service resides in the directory ws1. So from its parent, I issue

{% highlight shell %}
$ docker build -t users:get ws1/
{% endhighlight %}

And then run it with 
{% highlight shell %}
$ docker run -ti --network=host --name GetUserService  users:get
{% endhighlight %}

Notice that I use the most direct network configuration, this is certainly not your choice for production.

### Check user

The Dockerfile


{% highlight shell %}
FROM ubuntu:14.04
MAINTAINER ivan matmati
EXPOSE 5000
ENV PATH /opt/node/bin:$PATH
ADD node /opt/node
WORKDIR /logi/checkuserservice
RUN apt-get update
RUN apt-get -y install python
RUN apt-get -y install gcc
RUN apt-get -y install make
RUN apt-get -y install g++
ADD package.json index.js ./
RUN /opt/node/bin/npm install
CMD /opt/node/bin/node index.js
{% endhighlight %}

It's almost the same. The only noticeable difference is that I expose the port 5000 to call directly my service.

The code for index.js

{% highlight javascript%}
var seneca = require ("seneca")();

seneca.add("role:users,cmd:check", function (msg, respond) {

	//First get the user
	seneca.act({"role":"users","cmd":"get", "id":msg.id}, function (err, response) {

		if (err) return console.log (err)

			// Do check
			// ........
			response.user.checked=true;
			respond(null,response)
		})

}).use('mesh',{pin:"role:users,cmd:check"}).listen({port : 5000,pin:"role:users,cmd:check"})
{% endhighlight %}

Use the same package.json from above.
Now, let's build it and run it. The ws2 directory is my working space for this service.

{% highlight shell %}
$ docker build -t users:check ws2/
$ docker run -ti --network=host --name CheckUserService users:check
{% endhighlight %}

Now test it by 
{% highlight shell %}
$ curl -d '{"role":"users","cmd":"check","id":3}' http://127.0.0.1:5000/act
{% endhighlight %}

You should get the answer.

# Conclusion

Seneca plus Docker is a winning pair. They're both light and powerful. I only scratched the surface of this vast subject whether about Docker or Seneca.
I was pretty impressed by the quantity of plugins for Seneca. It seems to be a rational and viable choice that I discovered only lately.
But microservices embrace a lot more domains I didn't address here. To name a few : security, load balancing, transaction, etc.
I hope it gave you a taste of microservices and a first (naïve) architecture to illustrate the possibility offered. Your way is only beginning here.
The story is not over ...

# Reference

A few references obviously or not useful.

* [Seneca](http://senecajs.org/)
* [Getting started](http://senecajs.org/getting-started/)
* [Plugins list](http://senecajs.org/plugins/)
* [Nearform](http://www.nearform.com/seneca/)