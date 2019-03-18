---
title: Smartmeter Experiments part 1
layout: post
categories: [development]
tags: [smartmeter, p1]
---

Disclaimer: This is all based on the Dutch P1 Standard, in other countries it will simply not work or at best your experiences may vary

![Smartmeter](/images/smartmeter.png){: .align-right}

A smartmeter is a digital energy meter capable of sending its data and that of connected (modbus) devices to your energy provider  It also has a “P1 port” which provides the same information over a serial connection.

So it should be very easy to connect this to say .. a raspberry pi and indeed it is

![P1-USB](/images/p1_usb.png){: .align-left}

P1 to USB cables are readily available for under 20 euro's. don't try to make them yourself, the P1 has 5V levels, the R-Pi 3.3V.  which could blow up a chip on the R-Pi.

I installed [Raspbian Stretch Lite](https://www.raspberrypi.org/downloads/raspbian/) and burned that to an SD Card, I used a 4Gb one which is plenty.

to enable SSH, mount the SD to your desk/laptop and put an empty file called `ssh` into the `/boot` folder

to setup the wifi put the following into `/boot/wpa_supplicant.conf` replacing your network name and password

``` bash
country=NL
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
    ssid="NETWORK-NAME"
    psk="NETWORK-PASSWORD"
}
```

after you connect the R-Pi to the P1 port and boot it up. you should be able to ssh into it. 

When you look in the `/dev` folder you should see a `ttyUSB0` device.

Try to do a `cat /dev/ttyUSBO` and you would see something similar to this scroll in your console every x seconds (the Spec says 10 seconds, my meter at home does it every second)

``` bash
/XMX5LGBBFG10
 
1-3:0.2.8(42)
0-0:1.0.0(170108161107W)
0-0:96.1.1(4530303331303033303031363939353135)
1-0:1.8.1(002074.842*kWh)
1-0:1.8.2(000881.383*kWh)
1-0:2.8.1(000010.981*kWh)
1-0:2.8.2(000028.031*kWh)
0-0:96.14.0(0001)
1-0:1.7.0(00.494*kW)
1-0:2.7.0(00.000*kW)
0-0:96.7.21(00004)
0-0:96.7.9(00003)
1-0:99.97.0(1)(0-0:96.7.19)(160315184219W)(0000000310*s)
1-0:32.32.0(00000)
1-0:32.36.0(00000)
0-0:96.13.1()
0-0:96.13.0()
1-0:31.7.0(003*A)
1-0:21.7.0(00.494*kW)
1-0:22.7.0(00.000*kW)
0-1:24.1.0(003)
0-1:96.1.0(4730303139333430323231313938343135)
0-1:24.2.1(170108160000W)(01234.000*m3)
!D3B0
```

This is what is called a P1 Telegram and contains all the data collected by your smartmeter. in Part 2 of this blog series I will go into this format and see how we can parse this using Elixir

Now what you can do is install [Domoticz](http://www.domoticz.com/) and configure it to read the P1 connection through the ttyUSB0 device.

![Domoticz](/images/Domoticz.png)

If all you want is get insights in your energy usage this is all you need.