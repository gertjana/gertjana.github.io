---
layout: post
title:  "Mesos on a Raspberry Pi cluster"
date:   2016-05-11
categories: [development]
tags: [mesos, cluster, raspberrypi]
---
Sometimes seemingly unrelated events come together in a small project:

 - At the IoT Tech day this year in Utrecht I saw a couple of guys demonstrating a Raspberry Pi cluster running kubernetes. 
 - At my company we are currently investigating Mesos as a means to better manage our dockerized infrastructure.
 - I have a couple of raspberry pi's laying around

![](/images/Slack-for-iOS-Upload-1.jpg)

so lets get cooking

### Ingredients

 * a stackable [case](http://nl.aliexpress.com/item/1PCS-LOT-4-layer-Transparent-Acrylic-Case-Clear-Shell-Enclosure-with-Logo-for-Raspberry-Pi-2/32597229666.html?spm=2114.010208.3.1.0O5n0C&ws_ab_test=searchweb201556_9,searchweb201602_5_10017_10034_10021_507_10022_10020_10018_10019,searchweb201603_9&btsid=f31b93cc-3fa4-4d4d-ba5c-99e9277ac357)

 * a 5 port usb [powerhub](http://www.ianker.com/product/A2123122)
 * 4 short usb and 4 short network cables.

I already had an 8 port network switch and 4 Pi's laying around.

### Preparation
So i burned the latest [Hypriot](http://blog.hypriot.com/) (this image is basically Raspbian with docker installed) onto 4 8 Gb SD Card's
and changed the hostnames of my 4 pi's to 
``` bash
cluster-master
cluster-slave1
cluster-slave2
cluster-slave3
```
On each pi I also ran `sudo apt-get update;sudo apt-get upgrade`, expanded the filesystem, set the locale/timezone changed the password etc etc  

### Baking the master pi
Then I followed the steps on this page:
https://github.com/rpi-cloud/mesos-on-arm-docs

before starting the configure/make steps, i had to install 2 more dependencies:

``` bash
apt-get install libcurl-dev libsasl-dev
```

and for the java support java 8 and maven
make sure you export `JAVA_HOME` and `MAVEN_HOME`

also configure found a problem with the python version installed for now i solved that by disabling Python support and ran `./configure --disable-python`

You can disregard the last paragraph about it not working as you will be compiling with a patch for it earlier on

next to the steps on the page please clean up the pi as much as possible (deleting downloaded archives, apt-get autoremove) to avoid 'no space left on device' errors 

After compilation is finished, go to the build directory and start the master and a slave: (I'm starting them up a bit different then the page explains)

`ifconfig eth0 | grep inet` will give you the external ip of your pi


``` bash
./bin/mesos-master.sh --ip=<external-ip> --work_dir=/var/lib/mesos
./bin/mesos-slave.sh --master=<external-ip>:5050
```

and goto `http://cluster-master:5050` you will see the mesos home page with 1 slave connected

once you verified it's workings, create an archive of the build dir and copy it to the other pi's 

### Baking the slave Pi's

On your other Pi's install the dependencies:

``` bash
sudo apt-get install dh-autoreconf gcc-4.8 g++-4.8 cpp-4.8 \
  libapr1-dev libsvn-dev python-dev
```

and unpack the copied archive.
in `bin/mesos-slave-flags.sh` update the path to the launcher as it will be set to the directory where it was compiled

look at the output of `ifconfig eth0` to find the external ip of the slave and run the slave:
``` bash
./bin/mesos-slave.sh --master=<external-ip-of-master>:5050 --ip=<external-ip-of-slave>
```

the `--ip` is important otherwise the slave will bind to `127.0.1.1` and wont the master wont be able to connect to it

if you go to `http://cluster-master:5050` you should see 4 slaves connected now, and the total resources available be something like 16 cpu's and 1.8 Gb of memory

![](/images/Slack-for-iOS-Upload-2.jpg)

So far part 1 in the next parts i'll be covering

* make them start up at boot
* create and install a framework
* Have it do some work