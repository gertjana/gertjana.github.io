---
layout: post
title:  "Dockerify development"
date:   2014-10-09
categories: [development]
tags: [docker]
---
[Docker](http://www.docker.com) is a way to build your applications as [microservices](http://en.wikipedia.org/wiki/Microservices) and ship and run them as such.

As a first step I thought to make all the databases (mysql and postgres) I use for development docker containers so I can easily start and stop them, create copies and snapshots for dev/test/prod environments 

for the mac the easiest way to get started is by using [boot2docker](https://docs.docker.com/installation/mac/)

once you followed the installation steps there you are good to go

start it up by executing `boot2docker up` in a command shell
once that is done and you've created the `DOCKER_HOST` environment variable you can start running containers

a container contains of an vm image which you can run, modify, save and upload again

All images are hosted on https://registry.hub.docker.com/ 

As we are going to use mysql and postgres we need to download these images
``` bash
docker pull mysql
docker pull postgres
```

now we can run them (if you didnt do the pull, it will do it when you try to run them)
``` bash
docker run -e MYSQL_ROOT_PASSWORD=secret -d --expose 3306 --name mysql mysql
docker run -d --expose 5432 --name postgres postgres
```
this will startup the images, the -d runs them as a daemon, the --expose tells docker to expose the port to the host.

now these db's are available to be used in other docker containers.
for instance if you want to run phpmyadmin to admin your mysql you can run:
``` bash
docker run -e MYSQL_USERNAME=root -e MYSQL_PASSWORD=secret -p 80 --link=mysql:mysql corbinu/docker-phpmyadmin
```
here you specified that you want to link this container to your mysql one, and docker makes sure the 2 can talk to each other, also as I didn't specify a name docker will make one up for it (in this case `jolly_thompson` as you can see when running `docker ps`)
``` bash
docker@boot2docker:~$ docker ps
CONTAINER ID        IMAGE                              COMMAND                CREATED             STATUS              PORTS                   NAMES
c31afa9e5cac        corbinu/docker-phpmyadmin:latest   "/bin/sh -c phpmyadm   26 seconds ago      Up 25 seconds       0.0.0.0:49153->80/tcp   jolly_thompson               
bcd81ff32125        postgres:9                         "/docker-entrypoint.   2 hours ago         Up 2 hours          5432/tcp                postgres                     
498c394f149d        mysql:5                            "/entrypoint.sh mysq   2 hours ago         Up 2 hours          3306/tcp                jolly_thompson/mysql,mysql   

```


now if you would want to access the containers from other applications/tools there is a problem as the containers are running inside the boot2docker virtual machine

So any ports you want access to you need to forward.
There are several options for this.

First i tried adding port forwarding rules to the VM
```
VBoxManage modifyvm "boot2docker-vm" --natpf1 "postgres,tcp,127.0.0.1,5432,5432"
```
but these would require restarting the boot2docker host and sometimes it wouldn't and then i'd have to manually remove the rules and do a hard reset.

Then I realized boot2docker has an ssh command to open a shell into the host. I could use tunnelling as the ports are exposed to the host.

``` bash
boot2docker ssh -L 3306:localhost:3306 -L 5432:localhost:5432
```
and it works like a charm, for all the other applications it's like having native databases running

Shutting it all down is executing the following commands (run docker ps first to see what name the phpmyadmin has):
```	bash
docker stop mysql
docker stop postgres
docker stop jolly_thompson
docker rm mysql
docker rm postgres
docker rm jolly_thompson

boot2docker down
```

I combined all the steps above into 2 bash scripts, making it very easy to start/stop my environment

Next steps would be to have my applications build as docker containers removing the need for the port forwarding tricks, but that is for a later post