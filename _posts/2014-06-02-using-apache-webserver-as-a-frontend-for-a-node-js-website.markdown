---
layout: post
title:  "Apache webserver as frontend for a node.js website"
date:   2014-06-02
categories: [development]
tags: [apache, http, node]
---As I moved my blog from wordpress to node.js I ofcourse wanted to run it on port 80, as I already have apache running on my server using that as a frontend was obvious.

To get it running I needed to enable the proxy and the http handler for it:

``` bash 
a2enmod proxy
a2enmod proxy_http
```

Then I just changed the configuration to proxy to the host/port my node.js application is using:

``` xml 
&lt;VirtualHost *:80>
    ServerAdmin webmaster@localhost
    ServerName blog.addictivesoftware.net
    ProxyRequests off
    &lt;Proxy *>
        Order deny,allow
        Allow from all
    &lt;/Proxy>
    &lt;Location />
        ProxyPass http://blog.addictivesoftware.net:4000/
        ProxyPassReverse http://blog.addictivesoftware.net:4000/
    &lt;/Location>
&lt;/VirtualHost>
```

Restarting apache:

``` bash 
/etc/init.d/apache2 restart
```

and that's it
