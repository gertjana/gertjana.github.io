---
layout: post
title:  "Small proxy server in node.js"
date:   2013-09-27
categories: [development]
tags: [node, proxy, tip]
---
If at some point you need a very simple proxy server save the next 19 lines to a file (f.i. proxy.js)

``` javascript -->
var url = require('url');
var http = require('http');

http.createServer().listen(9000).on('request', function(request, response) {
  try {
    var options = url.parse(request.url);
    options.headers = request.headers;
    options.method = request.method;
    options.agent = false;

    var connector = http.request(options, function(serverResponse) {
            response.writeHeader(serverResponse.statusCode, serverResponse.headers);
            serverResponse.pipe(response, {end:true});
    });
    request.pipe(connector, {end:true});    
  } catch (error) {
    console.log("ERR:" + error);
  }
});
```

Install <a href="http://nodejs.org/">node.js</a> and run

``` bash 
node proxy.js
```