---
title: Light automation - switching lights with cron and python
date: 2022-08-06
categories: [Home Automation]
tags: [Home Automation, DIY]
img_path: /assets/img/smart-lights
---

![classic clock screen](cat-light-switch.gif)

I have a led strip behind my bed which I like to have turned on approximately one hour before I go to sleep.
Having to click a switch to turn them on and then again after an hour to turn it off is not the end of the world. But I thought that I could do better than that.
I have Yeelight Lighstrip Plus similar to [this one](https://www.amazon.co.uk/YEELIGHT-Homekit-Changing-Segmented-Control/dp/B09VSPTN12/ref=sr_1_1_sspa?crid=MN6L4U5TVFSF&keywords=yeelight+light+strip&qid=1659783420&sprefix=yeelight+light+stri%2Caps%2C92&sr=8-1-spons&psc=1&spLa=ZW5jcnlwdGVkUXVhbGlmaWVyPUExR1BHTFFLUjdQQ1VXJmVuY3J5cHRlZElkPUEwNDUwMjcyMk9EWFg4SlpSSjlSVSZlbmNyeXB0ZWRBZElkPUEwODIzNTQxWlVEV1lPWUtBTkhKJndpZGdldE5hbWU9c3BfYXRmJmFjdGlvbj1jbGlja1JlZGlyZWN0JmRvTm90TG9nQ2xpY2s9dHJ1ZQ==). When I bought them I read that they have an api that you can play with, so I decided to find more about it.

After quick search I found that there is a [Python library](https://yeelight.readthedocs.io/en/latest/) that allows to control Yeelight lights.
At that time I knew that with that I should be able to have full control over my led strip, so I decided to automate the entire process, and now I'd like to share my journey with you.

*Note: If you have lights from different provider than Yeelight, don't worry. Many of them have similar libraries. [Here](https://github.com/studioimaginaire/phue) is an example of a library for Philips Hue system.*

## Discovering leds

*Note: this part might be a bit different for each type of lights provider, it should be doccumented in each library / in the provider docs how to disover your led strip.*

After going through the library I noticed the following line of code:

```python
from yeelight import Bulb
bulb = Bulb("192.168.0.19")
```

It seems that I will be able to control my leds with `bulb` object, but how do I know the local ip address of my bulb 🤔.

The library says that there is a function called `discover_bulbs()` which can help with that, so let's try it out:

![](yeelight-fail.png)

Well, don't forget to install the library istelf as I did, let's try again, but now after: `pip install yeelight`.

![](yeelight-success.png)

This time it did work. I can see that there is an array of JSON objects. In my case there are two objects both of them have `'model': 'stripe'`, which is correct as I have two led stripes in my house. And what is more important each of them has `'ip': '...'` set to some value.

*Note: If `discover_bulbs()` didn't work for you ensure that you enabled [Lan Control](https://www.yeelight.com/faqs/lan_control) for your device.*

Now I only need to find out which one of them are my bed leds. I do that by simply trying to turn them on programatically like this:

```python
from yeelight import Bulb
bulb = Bulb("ip_of_the_first_led_strip")
bulb.turn_on()
```

## Writing a script

Now that I have the local ip address of my leds I can start writing a script that will turn them on and off. I start with a simple function that will turn them on then wait for given time and turn them off:

```python
def adjust_time(bulb, end_in_hours = 1, end_in_minutes=0):
    bulb.turn_on()
    sleep(end_in_hours * 60 + end_in_minutes)
    bulb.turn_off()
```

It works just fine, but I thought that it will be cool to lower the brightness as time flies. So when the leds turn on it is set to 100% and then it gradually lowers until leds turn off, after small modifications `adjust_time` looks like this:

```python
def adjust_time(bulb, end_in_hours = 1, end_in_minutes=0):
    start_time = datetime.now()
    end_time = now + timedelta(hours=end_in_hours, minutes=end_in_minutes)

    sleep_time = 30 #30s

    now = start_time
    bulb.turn_on()
    while(now < end_time):
        now = datetime.now()

        diff_in_minutes = (now - start_time).seconds / 60
        total_end_in_minutes = 60 * end_in_hours + end_in_minutes
        brightness = (1 - (diff_in_minutes / total_end_in_minutes)) * 100

        if brightness < 1:
            brightness = 1
        bulb.set_brightness(brightness)
        time.sleep(sleep_time)
    
    bulb.turn_off() 
```

Right now this function will adjust brightness each 30s. The brightness level will start from 100 and it will go down until 1, which is calculated here:

```python
diff_in_minutes = (now - start_time).seconds / 60
total_end_in_minutes = 60 * end_in_hours + end_in_minutes
brightness = (1 - (diff_in_minutes / total_end_in_minutes)) * 100
```

I didn't check the full documentation of `set_brightness` and I'm not sure how it will act when negative number is passed to it (it is possible now, because when we sleep for 30s it can happen that `diff_in_minutes / total_end_in_minutes > 0`), so I decided to play it safe and I added:

```python
if brightness < 1:
    brightness = 1
```

Let's try it out and see if it works. My final script looks like that:

```python
#!/usr/bin/python3

from datetime import datetime, timedelta
import time
from yeelight import Bulb

def adjust_time(bulb, end_in_hours=1, end_in_minutes=0):
    start_time = datetime.now()
    end_time = now + timedelta(hours=end_in_hours, minutes=end_in_minutes)

    sleep_time = 30 #30s

    now = start_time
    bulb.turn_on()
    while (now < end_time):
        now = datetime.now()

        diff_in_minutes = (now - start_time).seconds / 60
        total_end_in_minutes = 60 * end_in_hours + end_in_minutes
        brightness = (1 - (diff_in_minutes / total_end_in_minutes)) * 100

        if brightness < 1:
            brightness = 1
        bulb.set_brightness(brightness)
        time.sleep(sleep_time)

    bulb.turn_off()

bulb = Bulb('bulb_ip_address')
adjust_time(bulb, end_in_hours=0, end_in_minutes=5)
```

I can test it now:

```
chmod +x adjust_lights.py
./adjust_lights.py
```

💡  And it works!

## Adding a cron job

Cool, I have my script ready, now instead of clicking the switch I have to turn my script on an hour before going to bed.

![insert roll safe gif](roll-safe.gif)

But do I really have to?

### Cron job to the rescue

What is Cron?

> Cron is a job scheduling utility present in Unix like systems. The crond daemon enables cron functionality and runs in background. The cron reads the crontab (cron tables) for running predefined scripts.

Seems like the perfect tool for the job.

One more thing before adding a cron job, remember when testing my script I called
`adjust_time(bulb, end_in_hours=0, end_in_minutes=5)` with that settings my lights would be on only for 5 minutes. So before adding the job I changed it to `adjust_time(bulb, 1, 0)`. You can use values that you prefer, and you can always change them later. All cron does is running your script at given time, so you can change it whenever you want.

To add a job to the cron open your terminal and type `crontab -e`. It will either open automatically, or will ask you for the editor you want to open it in, or will show you the permission error. If you don't have permissions try running it in sudo mode.

After openning the crontab, add a new job to it. To do that you have to find a path to your script (you can use `pwd` while in the directory of your script). When you know the path, add the following line at the end of the crontab:

`0 22 * * * /path-to-the-script/adjust_lights.py`

It means that cron will run this script everyday at 22:00, you can set the hour that suits your own needs here.

![That's all folks](thats-all-folks.jpeg)

And that's it. My leds are now automatically turning on each day at 22:00, no need to click the switches. Just remember to have the machine with your cron on. I use my Raspberry Pi for it as I almost never turn it off.


