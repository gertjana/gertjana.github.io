---
layout: post
title:  "Simple HTTP Server"
date:   2014-02-27
categories: [development]
tags: [http, python, tip]
---
If you ever just need a simple http server to serve up some html/javascript, because for instance access to local filesystem, camera, microphone etc is disabled when using the file:// protocol

just install Python 3.x and run

``` bash
python -m http.server 8000
```
from the command line and voila

