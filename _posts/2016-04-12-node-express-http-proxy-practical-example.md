---
layout: post
title: "Node.js Express http proxy simple practical example."
date: 2016-04-12 09:25:00 +0100
comments: true
---


Imagine that we want to add new functionality to an old application written in java, and we would like to use Node.js. 

Maybe we have our programming in Node.js already done and now we wanted to deploy the application, and run in parallel with the old java application.

The old app listens on port `7002`, the new one should listen on different port, lets say `7003`. 

But there is a firewall that allows access only to the old `7002` port, so we are not able to reach the new application from the browser. How to solve it?

There is one quite obvious solution, we should just ask admins to open new port in the firewall. And we can be perfectly happy with that. Unfortunately, that is a kind of task, that would summon a lot of unnecessary "why" questions (e.g. *Why did you not use the old java api?*). 

We can save our time if we simply **proxy the old application in a new one**. 

What are the prerequisites? Firstly, because there is only `7002` port allowed in the firewall, *we should be able to switch the listening port of the old application* to some other port and then use our precious (firewall already opened) `7002` port in the new Node.js application. 

```
             
| old app |           | new Node.js |           |         |
| running | <-------> | application | <-------> | Browser |
| on other|           | running on  |           |         |
| port    |           | 7002 port   |                    
                      | (opened on  |
                      |  firewall)  |

```
If we are not able to switch the listening port of the old application, the proxy won`t work.

We also need Node.js installation on the target system. Please look at my previous post [How to install node.js and npm without sudo and curl on remote Linux.](/2016/03/how-to-install-nodejs-and-npm-without-sudo-on-remote-linux)


Main Idea
=========

We would use `express-http-proxy` for the most of our http proxy, but for  continuous content feeding, i.e. if we need to watch the progress in the browser, and for large files downloads we are going to use different approach. We are also not forgetting to make routes to our new application.


Programming
============

In this section I am going to explain just the middleware code in `server.js` file. That means, all other stuff is mentioned very briefly. You should already know how to download necessary `Node.js` modules, i.e. `npm` and/or `bower` tools for `Node.js` dependency management. The front-end code for the new application is mentioned here just because the proxy works together with the new application.

#### Directory structure
```             
├── public
│   ├── app.js
│   ├── bower_components
│   │   └── angular
│   │       ├── README.md
│   │       ├── angular-csp.css
│   │       ├── angular.js
│   │       ├── angular.min.js
│   │       ├── angular.min.js.gzip
│   │       ├── angular.min.js.map
│   │       ├── bower.json
│   │       ├── index.js
│   │       └── package.json
│   └── index.html
└── server
    └── server.js    
```
Middleware code for the app router and the proxy is defined in `server.js` file. The folder `public` contains  front-end code for new application. `bower_components` for `Angular.js` were downloaded using `bower`. This is a standard file structure, that can be found in many projects using `express` and `angular` 

#### File "server.js"

We are focused on working with the code in this file. This file contains all middleware code for our small project, needed for routing and proxying.

##### Complete content of "server.js"

```             
/**
 *
 * Created by pavelnovotny on 07.04.16.
 */

var express = require("express");
var path = require("path");
var proxy = require('express-http-proxy');
var cors = require("cors");
var app = express();
var request = require("request");

app.use(cors());

//app
app.get("/app", function(req, res) {
    res.sendFile(path.join(__dirname, '../public', 'index.html'));
});
app.use(express.static('public'));

//proxy that does not wait for response finish (feeds browser immediatelly)
app.get('/processCommand', function(req,res) {
    var newurl = 'http://old-application-address:7002' + req.originalUrl;
    request(newurl).pipe(res);
});
app.get('/download', function(req,res) {
    var newurl = 'http://old-application-address:7002' + req.originalUrl;
    request(newurl).pipe(res);
});

//standard proxy
app.use('/', proxy('old-application-address:7002', {
    forwardPath: function(req, res) {
        return require('url').parse(req.url).path;
    },
    intercept: function(rsp, data, req, res, callback) {
        // rsp - original response from the target
        //to get rid of express application/octet-stream
        if (rsp.headers['Content-Type'] === undefined) {
            res.setHeader('Content-Type', 'text/html; charset=UTF-8');
        }
        callback(null, data);
    },
    decorateRequest: function(req) {
        //sometimes are form post data sent incorrectly
        bodyContent = req.bodyContent.toString();
        req.bodyContent=bodyContent;
        return req;
    }
}));

app.listen(7003);

```



#### Detailed explanation of routes and proxy

##### Route to new application

Beside our proxy, there is the new application, that works together on the same address. We use standard `express.js` routing:


```             
app.get("/app", function(req, res) {
    res.sendFile(path.join(__dirname, '../public', 'index.html'));
});
app.use(express.static('public'));
```
In that way, `/app` will download `index.html`. All resources that reference `public` folder in the index.html are allowed to download as well. 


##### Proxy for continuous content feed

Standard behaviour of `express-http-proxy` is that it waits, until the response  of proxied resource is complete. It can be a problem, when the resource is large, e.g. ~100MB+ file downloads (memory consumption), or if the browser renders "on fly" received data (there are subjectively late responses, no "on fly" check of the data is available).

For both cases we can use `pipe` on `request`.

```             
//proxy that does not wait for response finish, feeds browser immediatelly
app.get('/processCommand', function(req,res) {
    var newurl = 'http://old-application-address:7002' + req.originalUrl;
    request(newurl).pipe(res);
});
app.get('/download', function(req,res) {
    var newurl = 'http://old-application-address:7002' + req.originalUrl;
    request(newurl).pipe(res);
});

```
In the proxied application, `/processCommand` sends data about current running command. The data are sent continually during the duration of the command, which can last up to several minutes. The browser renders the data, so the user is able to check what is current state of the command. On the other hand the `/download` request returns possibly very large data. There is no need to cache it in the standard `express-http-proxy`.


##### Standard express-http-proxy

For all other cases we can benefit from easy `express-http-proxy` setup.

```             
//standard proxy
app.use('/', proxy('old-application-address:7002', {
    forwardPath: function(req, res) {
        return require('url').parse(req.url).path;
    },
    intercept: function(rsp, data, req, res, callback) {
        // rsp - original response from the target
        //to get rid of express application/octet-stream
        if (rsp.headers['Content-Type'] === undefined) {
            res.setHeader('Content-Type', 'text/html; charset=UTF-8');
        }
        callback(null, data);
    },
    decorateRequest: function(req) {
        //sometimes are form post data sent incorrectly
        bodyContent = req.bodyContent.toString();
        req.bodyContent=bodyContent;
        return req;
    }
}));

```
Why do we need `intercept` and `decorateRequest`? Well, our old application is not perfect. It for example sometimes does not fill `Content-Type` http header. For the browser it is not a problem, as it treats it like `text/plain` when missing. But `express.js` thinks better. It treats it like `application/octet-stream` and sends such header to the browser. As a result, instead of rendering the page, the browser downloads it as a file. That's the reason we must treat such cases manually:

``` 
        //to get rid of express application/octet-stream
        if (rsp.headers['Content-Type'] === undefined) {
            res.setHeader('Content-Type', 'text/html; charset=UTF-8');
        }

```

`express-http-proxy` also has problems with standard form request data, they were corrupted after proxying. I did not seek exact reason, but used this workaround, that works well:

``` 
        //sometimes are form post data sent incorrectly
        bodyContent = req.bodyContent.toString();
        req.bodyContent=bodyContent;
        return req;

```

That`s all. Simple proxy an old application when no other port is available, and you want new functionality in Node.js.
