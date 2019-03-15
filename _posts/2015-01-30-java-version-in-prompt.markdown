---
layout: post
title:  "Java version in prompt"
date:   2015-01-30
categories: [development]
tags: [bash, fish, tip, java]
---
I already have the current directory and my git state in my prompt which helps me in my daily development work

Last week I ran into a bit of problem as apple updated my java version behind the scenes (I did tell it could do it btw) and this made jenv (which i use to manage multiple java versions) misbehave.

Long story short, I managed to publish one of our oss scala libraries compiled with java 8 instead of java 7.
which gave weird errors in any scala 2.10 application which was using that library.

So lets put the java version in the prompt too so I won't make that mistake again ;)

The thing to add to your PS1 env var (if bash is your shell of choice is
``` bash
java -version 2>&1 sed -n '1p' | sed 's/^.*\"\(.*\)\..*\"/\1/g'
```
this will output 1.7 for Java 7. the redirect `2>&1` is needed as java -version outputs on stderr 

I'm using fish shell and there just add tot he fish_prompt function in `~/.config/fish/config.fish`
``` bash
java -version ^| sed -n '1p' | sed 's/^.*\"\(.*\)\..*\"/\1/g'
```
A nice feature of fish shell is that it can pipe on stderr too by using `^|`

