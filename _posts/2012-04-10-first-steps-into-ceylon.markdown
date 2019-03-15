---
layout: post
title:  "First steps into Ceylon"
date:   2012-04-10
categories: [development]
tags: [ceylon, languages]
---
A good friend of mine is part of the <a href="http://www.ceylon-lang.org/">ceylon</a> development team, so when M2 came out which had Java interoperability, I decided to have a go and dabble a bit with it and mis-use our friendship by badgering him with questions, but he'll get something out of it too. well mostly bugs to fix (and the start of an <a href="https://github.com/ceylon/ceylon-lang.org/wiki/FAQ_proposals" target="_blank">FAQ</a>).

Around the same time I was looking into <a href="http://netty.io">netty</a> and decided to port the small webserver I made with netty to ceylon.

During this exercise I ran into a few blocking issues, which is normal in this stage of the project so I will not mention those as they are fixed by the time you'll be reading this.

First let setup the project (I already have a maven based java project, I'm adding my ceylon sources to /src/main/ceylon, but i'm compiling from the commandline):

As we have a dependency on netty, for now we need to copy it in our local ceylon repository
<ul>
	<li>Create the required directory structure
``` bash 
~/.ceylon/repo/org/jboss/netty/3.4.0/
```
</li>
	<li>copy your downloaded netty jar in here and rename it to org.jboss.netty-3.4.0.jar</li>
	<li>create a sha1 checksum by calling (As sha1sum is not part of the MacOS tools, I had to build it from <a href="http://www.microbrew.org/tools/md5sha1sum/">sources</a>)
``` bash 
sha1sum org.jboss.netty-3.4.0.jar &gt; org.jboss.netty-3.4.0.jar.sha1
```
    </li>
</ul>

And import it  in the module.ceylon

``` ceylon
Module module {
    name='net.addictivesoftware.nbws';
    version='0.1';
    dependencies = {
       Import {
           name = 'org.jboss.netty';
           version = '3.4.0';
       }
    };
}
```

Next lets have a look at the main class:

The Java version is on <a href="https://github.com/gertjana/nb-ws/blob/master/src/main/java/net/addictivesoftware/nbws/http/HttpServer.java" target="_blank">github</a> (will open in a new window/tab for comparing)

``` ceylon
import org.jboss.netty.bootstrap {ServerBootstrap, Bootstrap}
import org.jboss.netty.channel.socket.nio {NioServerSocketChannelFactory}
import java.net {InetSocketAddress}
import java.util.concurrent {Executors{newCachedThreadPool}}

variable Integer port := 9000;

shared void run() {
    String[] args = process.arguments;
    if ((nonempty args) &amp;&amp; args.size == 1) {
        Integer p = parseInteger(args.first ? "-1") ? -1;
        if (p != -1) {
             port := p;
        }
        print("Starting server on port: " + port);
        ServerBootstrap bootstrap = ServerBootstrap(
            NioServerSocketChannelFactory(
                newCachedThreadPool(),
                newCachedThreadPool()));

        bootstrap.pipelineFactory := HttpServerPipelineFactory();
        bootstrap.bind(InetSocketAddress(port));
        print("Server started");
    } else {
        print("specify port as the only argument");
    }
}
```
First thing you will notice is the imports. particular the static method import of newCachedThreadPool(), by specifying it as

`import your.package {Class{method}}
`
the method becomes a top-level method, optionally you can give it a name by doing

`import your.package {Class{name=method}}`

In ceylon there is no new keyword, when porting java code, just remove the new and you'll be fine.

Commandline Arguments are passed on by process.arguments, the nonempty checks for null or empty, this construction in combination with the exists (not null) keyword makes for pretty readable code.

Something I didn't like, but maybe i'm overlooking something is the construction that parses the first argument to an Integer.

``` ceylon
Integer p = parseInteger(args.first ? "-1") ? -1;
if (p != -1) {
    port := p;
}
```

This is because args.first returns a String? (the ? meaning it can be null) which I need to check for null (with the ? operator) because the parseInteger requires a String (cannot be null in ceylon), and returns a Integer? which I then need to check for null again
This seems overly complex and is going to happen a lot as you tend to restrict your parameters to not be null and your return value to be null, also as Java objects can be null, they will be cast to their [object]? counterparts in ceylon.  But like I said I'm probably missing something obvious.

Next we need two more classes:

HttpServerPipelineFactory (<a href="https://github.com/gertjana/nb-ws/blob/master/src/main/java/net/addictivesoftware/nbws/http/HttpServerPipelineFactory.java" target="_blank">Java version</a>):

``` ceylon
import org.jboss.netty.handler.codec.http{HttpChunkAggregator,HttpRequestDecoder,HttpResponseEncoder}
import org.jboss.netty.handler.stream{ChunkedWriteHandler}
import org.jboss.netty.channel{ChannelPipeline,ChannelPipelineFactory,Channels{staticChannelPipeline=pipeline}}

shared class HttpServerPipelineFactory() satisfies ChannelPipelineFactory {
    shared actual ChannelPipeline pipeline = staticChannelPipeline();

    pipeline.addLast("decoder", HttpRequestDecoder());
    pipeline.addLast("aggregator", HttpChunkAggregator(65536));
    pipeline.addLast("encoder", HttpResponseEncoder());
    pipeline.addLast("chunkedWriter", ChunkedWriteHandler());
    pipeline.addLast("handler", HttpServerHandler());

}
```

In line 12 we add our own Handler that will handle the requests:

HttpServerHandler (<a href="https://github.com/gertjana/nb-ws/blob/master/src/main/java/net/addictivesoftware/nbws/http/HttpServerHandler.java" target="_blank">Java version</a> not complety converted to ceylon yet):

``` ceylon
import org.jboss.netty.channel{SimpleChannelUpstreamHandler, ChannelHandlerContext,
                        MessageEvent, Channel, ChannelFuture, ChannelFutureListener{close=iCLOSE}}
import org.jboss.netty.handler.codec.http{HttpRequest, DefaultHttpResponse, HttpResponse}
import org.jboss.netty.handler.codec.http{HttpVersion{http11=iHTTP_1_1}, HttpResponseStatus{ok=iOK}}
import org.jboss.netty.buffer{ChannelBuffers{copiedBuffer}}
import org.jboss.netty.handler.codec.http{HttpHeaders{isKeepAlive}}

shared class HttpServerHandler() extends SimpleChannelUpstreamHandler() {

    shared actual void messageReceived(ChannelHandlerContext? ctx, MessageEvent? e) {
        if (exists e) {
            if (is HttpRequest request=e.message) {
                HttpResponse response = DefaultHttpResponse(http11, ok);
                Channel ch = e.channel;
                ch.write(response);	

                String responseText = "Retrieved: " + request.uri;
                ChannelFuture writeFuture = ch.write(copiedBuffer(responseText, "UTF-8"));

                if (!isKeepAlive(request)) {
                    // Close the connection when the whole content is written out.
                    writeFuture.addListener(close);
                }
            }
        }
    }
}
```

Two things worth mentioning here:
1) The following structure:

`if (is HttpRequest request=e.message)`

will assign e.message to the request variable and checks if it is of type HttpRequest.
If this is true then inside the if, the request is of type HttpRequest, no casts necessary, I really like this.

2) at this point capitalized class names have a special meaning to the compiler, that is why you need to prefix it with i and assign it a name

`import org.jboss.netty.handler.codec.http{HttpVersion{http11=iHTTP_1_1}}`

To conclude, Ceylon allows for shorter and more readable code which for me means less bugs and better maintainability. The java interop still has some rough edges. but that will improve, and once the SDK is released, we will be less dependent on the java classes anyway

The entire project is hosted on <a href="https://github.com/gertjana/nb-ws">github</a> for those who want to have a closer look.
to compile the ceylon classes, run

`ceylonc -src src/main/ceylon net.addictivesoftware.nbws`

from the commandline.
It will need a compiler build after the 14th of April to work

to run it:

`ceylon net.addictivesoftware.nbws/0.1`

next I'll be looking into reading a file from a resource stream, doing some reflection and running unittests with junit