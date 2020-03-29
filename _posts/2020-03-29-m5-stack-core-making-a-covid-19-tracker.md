---
layout: post
title: M5 Stack Core - making a COVID-19 tracker
category: development
tags:
- arduino
---

Being confined to my house during the COVID-19 outbreak can get a bit boring, so I was very happy that my M5-Stack Core arrived.
![M5 Stack core grey](/images/m5stack.png){:style='width: 200px;float:right;'}
This is an ESP32 with a battery, 320x240 LCD screen, a SD card reader, 9-axis posture and accellerometer and a few buttons all in a compact case, ideal for rapid prototyping
so lets get started.

I use the Arduino IDE and the instructions to set it up for the M5 Stack can be found here:
https://docs.m5stack.com/#/en/arduino/arduino_development

After that you can already play around with dozens of examples that are included in the library
![M5 Open](/images/m5_open.png){:style='float:right;width:400px;'}

as this was the thing that was already on my mind I decided to display COVID-19 data, (confirmed, recovered, deaths) for the world and per country)

A quick search on google, gave me an endpoint at rapidapi.com that gave me exactly those numbers. registering an account there provided me with an apikey to get that data
so to start off, we need to setup:
* Serial, for debugging purposes
* Wifi, to be able to connect to the API
* M5 Library, to access the buttons and display

```c
#include <M5Stack.h>
#include <WiFi.h>

void setup(void) {
  Serial.begin(115200);
  while(!Serial);    // time to get serial running
  Serial.println("Corona Tracker");
    
  M5.begin(true, false, true); //lcd + serial enabled, sd card disabled
  M5.Power.begin(); // to turn the device off after a few minutes inactivity
  
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("");
  Serial.print("WiFi connected with IP address: ");
  Serial.println(WiFi.localIP());
}
```
now to read the buttons and set some variables

```c
const String countries[NR_COUNTRIES]  = {"gb","us","it", "es", "cn"};
int country_index = 0;
String scope = "World";
String path = "/totals";
bool refresh_http = true;

void loop() {
  M5.update(); // update button state
  
  if (M5.BtnA.read()) {
    scope = "World";
    path = "/totals";
    refresh_http = true;
  } else if (M5.BtnB.read()) {
    scope = "nl";
    path = "/country/code?code=nl";
    refresh_http = true;
  } else if (M5.BtnC.read()) {
    scope = String(countries[country_index]);
    path = String("/country/code?code=") + countries[country_index];
    country_index += 1;
    if (country_index >= NR_COUNTRIES) {
      country_index = 0;
    }
    refresh_http = true;
  }
	 ...
	 delay(100); // query the button's 10 times per second.
}
```
Ok, now pressing the buttons will set the path for the api call, and a scope var, which will be displayed

The `refresh_http` boolean is to get the data from the API as we will see later

we don't want to bombard the API with calls, as we only need to get the data when a button is pressed and maybe every minute or so

```c
const int REFRESH_RATE = 60; //seconds
int cnt_http = 0;

void loop() {
  ...
	
  cnt_http += 1;
  
  if (cnt_http > 10*REFRESH_RATE) {
    cnt_http = 0;
    refresh_http = true;
  }

  if (refresh_http) {
    refresh_http = false;
		
		... do the http call
		}
}
```
This code is called inside the loop, so every 100ms, the cnt is updated until it reaches (10*60 * 100ms) which is a minute and set's the refresh_http to true, now the http call is made and the refresh_htpp is set back to false. as you can above, also every press of the button toggles the refresh_http

Now i'm ready to get the data
```c
  if (refresh_http) {
    refresh_http = false;

		HTTPClient http;
    http.begin(String("https://") + host + path);
    http.addHeader("X-rapidapi-host", host);
    http.addHeader("X-rapidapi-key", apikey);
  
    int httpCode = http.GET();
   
    if (httpCode > 0) {
      String payload = http.getString();
      parse(payload);
    } else {
      Serial.println("Error on HTTP request");
    }
    
    update_display();

    http.end(); //Free the resources
}
```

The response i'm getting back is JSON, so I need to parse that:
An example response is:
```Json
[{"confirmed":"532238","recovered":"124327","deaths":"24086"}]
```

```c
#include "ArduinoJson.h"

StaticJsonDocument<300> doc;
int confirmed = 0;
int recovered = 0;
int deaths = 0;

void parse(String payload) {
  DeserializationError error = deserializeJson(doc, payload);
  
  confirmed = doc[0]["confirmed"];
  recovered = doc[0]["recovered"];
  deaths = doc[0]["deaths"];
}
```
Now all that is left is to display this on the lcd screen:
```c
void update_display() {
  M5.Lcd.fillScreen(TFT_BLACK); 
  M5.Lcd.setCursor(0, 0, 2);

  M5.Lcd.setTextColor(TFT_WHITE,TFT_BLACK);
  M5.Lcd.setTextFont(4);
  M5.Lcd.println(String("Corona Tracker - ") + scope);

  M5.Lcd.setTextFont(2);
  M5.Lcd.setTextColor(TFT_WHITE,TFT_BLACK);
  M5.Lcd.println("confirmed");
  M5.Lcd.setTextColor(TFT_YELLOW,TFT_BLACK);
  M5.Lcd.setTextFont(7);
  M5.Lcd.println(confirmed);
  
  M5.Lcd.setTextFont(2);
  M5.Lcd.setTextColor(TFT_WHITE,TFT_BLACK);
  M5.Lcd.println("recovered");
  M5.Lcd.setTextColor(TFT_GREEN,TFT_BLACK);
  M5.Lcd.setTextFont(7);
  M5.Lcd.println(recovered);
  
  M5.Lcd.setTextColor(TFT_WHITE,TFT_BLACK);
  M5.Lcd.setTextFont(2);
  M5.Lcd.println("deaths");
  M5.Lcd.setTextColor(TFT_RED,TFT_BLACK);
  M5.Lcd.setTextFont(7);
  M5.Lcd.println(deaths);
}
```
notable here is `M5.Lcd.setTextFont(7);` this is a font mimicing 7-segment displays

The end result is shown here ![Coronatracker](/images/Coronatracker.png){:style='float:right;'}

Conclusion is that the M5 Stack, makes rapid prototyping very easy and fast. It is also a very good tool for education purposes.

now go watch the video I made about this [https://youtu.be/ykVsMTuumOM](https://youtu.be/ykVsMTuumOM)

or have a look at the code here: [https://github.com/gertjana/M5_Corona_tracker](https://github.com/gertjana/M5_Corona_tracker) 

now order your own M5 Stack, stay safe in your home, practise social distancing and I will see you after it will be just another 'flu'
