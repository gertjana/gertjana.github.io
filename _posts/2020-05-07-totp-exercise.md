---
layout: post
title: TOTP exercise
category: development
tags:
- ''
- aws
- totp
- security
---

for obvious security reasons I have setup MFA (Multifactor authentication) on all accounts that support it. but this can be a bit cumbersome in some situations.

Especially when you are working with Serverless applications on AWS. I work a lot with Cloudformation and SAM (everything is Infrastructure as code) but that does mean that during the day I'll be typing in MFA codes a lot.

As an exercise I wanted to see if I could get those codes visible without having to resort to the Google Authenticator app.

now after trying and failing to make an ESP32 Device with a small screen showing those codes (library does not seem to work, or can't deal with the longer secret AWS uses), and micropython missing required libraries, I had another idea.

What if I could have that code show up in my touchbar, I know, I know MFA needs to be another device! 
so a disclaimer:

> What is described below will lower the security of your system and your AWS account: do so at your own risk

but if i would 
 - need sudo to enable it and disable it automatically after a certain amount of time
 - only shows it in the touchbar after pressing a secret key combination
 - store the secrets in the keychain

I would say I didn't decrease my security level by much. but I gained some productivity.

I'm already using a tool called BetterTouchTool that allows customisation of the touchbar on my mac. 

so first of all how to do we get that code?

Whenever you add a application to for instance Google Authenticator, you have to scan a QR Code. this QR Code contains a secret
if you would scan that with a normal qr code scanner you will get this url with a secret param:
```
otpauth://totp/Amazon%20Web%20Services:[account]?secret=[secret]&issuer=Amazon%20Web%20Services
```
now here you can see that it uses TOTP (Timebase One Time Password) which is an extenstion to HOTP (HMAC-Based One Time Password)

to summarize really quickly:

HOTP uses a hash algorythm to create a digest from the secret and a counter)
TOTP introduces a time based component for the counter part  (the counter is x second (default is 30) steps counted from the epoch) 

Now Python has a OTP library that can create the MFA code from that secret.
```python
#!/usr/bin/env python3

import pyotp
import sys

secret = sys.argv[1]
totp = pyotp.TOTP(secret)
print(totp.now())
```
This wil when you give it a secret return the 6 digit code

To not have the secret in my scripts I've added it to the keychain with

```bash
security add-generic-password -a [account] -s [service] -w [secret]
```

and retrieve it again in a bash script
```bash
#!/bin/bash
me=`whoami`
secret=`security find-generic-password -a $me -s [service] -w`
echo `python3 [path]/get_code.py $secret`
```
so now I have something I can call from BetterTouchTool:

In BetterTouchTool you can create a button (widget) and then run an applescript when started and another one when you press the button, you can also define a key combination to only show it when pressed.

so when starting it It will run this applescript: it will execute the script and set the text property with the result to have it shown as the button text

```applescript
set code to do shell script "/Users/[account]/btt_scripts/code_aws.sh"
return "{\"text\":\"" & code & "\"}"
```

when pressed it get the latest code again but now put the code on the clipboard

```applescript
set code to do shell script "/Users/[account]/btt_scripts/code_aws.sh"
set the clipboard to code as text
```

To conclude with the pytotp libray it is trivial to build your own 'google authenticator' application.

Now it works I will remove it again!

And focus on getting this to run on an ESP32. although this will probably mean I have to port the python library to C, as I haven't found any libraries that work with the AWS secret, just some abandoned projects that tried to do the same.
