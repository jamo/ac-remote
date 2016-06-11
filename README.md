# Configs and setup instructions for my Air Conditioning unit  / ASHP (Air source heat pump)

## Background

The device comes by default with only an IR remote control. And the manufacturer provides way overpriced remote controller that allows you to control it via sms/internet (not really sure).
Anyways, I wanted to control the device remotely, and even maybe automatically change between the maintenance mode, heating mode and cooldown mode.

Since I already have Raspberry PI setup for video surveillance in the same location it made it to a perfect platform to use to control my AC unit.

There are a bunch of tutorials on how to get it done, and some even starting to reverse engineer the IR protocol of their devices.
However, I didn't find an perfect source and thus it took some more research to get it done. In retrospect, I probably tried to make it even harder than what was necessary.
Thus I'm documenting here most of the workflow.

Also, unlike your traditional TV remote, the remote for my AC unit sends all of the settings in each message, making knowing the state easier, but causing the traditional tools (like the lirc autolearn) to fail.

## Setup

### Hardware

My platform of choice was the Raspberry PI 3; I've also successfully tried this with the Raspberry PI 2 model b.

I purchased the [IR leds](https://www.sparkfun.com/products/9349) and the [IR receiver](https://www.sparkfun.com/products/10266) from [Sparkfun](https://www.sparkfun.com/). Just make sure to get the correct wavelength for your device. Most of the user manuals and repair manuals available freely mentions the wavelength used. In my case it was close enough for 950nm.

### Software

I used the popular [Linux Infrared Remote Control](http://www.lirc.org/) for recording and transmitting the IR signals.

After installing the `lirc` package I needed to enable it in two places, in
`/etc/modules`  and in `/boot/config.txt` In both I also needed to set the GPIO pins used for reading and transmitting the signals (see snippets below)

##### /etc/modules

```
lirc_dev
lirc_rpi gpio_in_pin=23 gpio_out_pin=22
```

##### /boot/config.txt

```
dtoverlay=lirc-rpi,gpio_in_pin=23,gpio_out_pin=22
```

Now with the lirc properly configured (and after restarting the raspberry pi), check from dmesg that the `lirc` module got downloaded, it should look similar to this:
```bash
$  dmesg | grep lirc
[    5.406585] lirc_dev: IR Remote Control driver registered, major 246
[    5.600078] lirc_rpi: module is from the staging directory, the quality is unknown, you have been warned.
[    6.770931] lirc_rpi: auto-detected active high receiver on GPIO pin 23
[    6.954146] lirc_rpi lirc_rpi: lirc_dev: driver lirc_rpi registered at minor = 0
[    7.202830] lirc_rpi: driver registered!
```

##### /etc/lirc/hardware.conf

For lirc you'll need to set the driver, the device to use and the module:

The defaults should be mainly ok, these are the main ones I had to fiddle with.

```
DRIVER="default"
# usually /dev/lirc0 is the correct setting for systems using udev
DEVICE="/dev/lirc0"
MODULES="lirc_rpi"
```


### Wiring it all up with

Just simply connect the IR receiver (as per it's spec) to +, - and the data to the pin you configured to be the input, in my case pin 23.

And the led (with properly sized resistor) to ground and the pin set for output, in my case pin 22.

*TODO: add diagram*

### Testing the IR receiver

Firstly let's confirm that we can indeed read from the IR receiver.
Let's start by making sure the lirc daemon is not running `service lirc stop`.
Now we can read `sudo mode2 -raw /dev/lirc0`
and shoot it with IR signals from your remote. You should see a bunch of numbers to show up. That's the IR command sent be the remote, like the snippet below. With my remote the message consisted in average of 50 rows.

```
    4362     4475      586     1620      582     1622
    579     1621      582     1563      587      456
    573      504      586     1551      579      475
    585      466      593      453      586      458
    581      540      551     1613      579     1568
    <snip>
```

Now we have everything set up to record all of commands.

### Recording the commands and saving them for later use

Now this is only in the scenario where all of the settings for the device are transmitted over in the single message. So this would be different for your traditional TV remote, that will only transmit a single command at once (like volume up command has no information about the channel and screen brightness etc)

But the remote for the AC I have sends always all of the settings.

So let's begin.

Firstly I would suggest having the terminal running the `sudo mode2 -raw /dev/lirc0` command and a text editor open side by side. As we are rather manually constructing the configuration. - We want to record the message for each of the settings.

In my case, after a few weeks of the AC usage, I had determined that we mainly wanted to use the following settings (out of all of the possible conditions):

mode | fan speed          | temp
-----|--------------------|-----
heat | auto               | 23
heat | auto               | 20
heat | auto               | 18
heat | full               | 23
heat | full               | 20
dry  | auto (only option) | 23
dry  | auto (only option) | 20
dry  | auto (only option) | 18

So let's record the message for each of the settings.

Basically you should ignore the possible messages transmitted while setting the remote to settings you want to save, except the last one. That should now contain all of the settings (heat, speed of fan, temperature, etc.)

Now copypaste it over to your texteditor in the following format

```
name NAME_OF_THE_SETTING
    the IR raw message as it appears from the `mode2` output
    spanning over multiple rows
```

e.g:
```
name HEAT_AUTO_23
    4362     4475      586     1620      582     1622
    579     1621      582     1563      587      456
    573      504      586     1551      579      475
    585      466      593      453      586      458
    581      540      551     1613      579     1568
    <snip>
```

And now repeat this for all of the settings you want to use. You also may come back to this and add more settings later on.


Now with the config file in your editor let's add rest of the configs for lirc.

I used PI as name of the remote, set it to whatever, but we'll need to later when we start sending the IR packets.

The other values will depend on your device, mainly the frequency. You can try researching for proper values, I found the frequency from the repair manual for my particular device. And others with `irrecord` (which I'm not intending to cover)

```
begin remote

  name  PI
  flags RAW_CODES
  eps            30
  aeps          100

  gap          450
  frequency    38000

      begin raw_codes
        name HEAT_AUTO_23
             4362     4475      586     1620      582     1622
              579     1621      582     1563      587      456
              573      504      586     1551      579      475
              585      466      593      453      586      458
              581      540      551     1613      579     1568
              <snip>
      end raw_codes
end remote
```

You can view my full config [here](https://github.com/jamox/ac-remote/blob/master/unit1/etc/lirc/lircd.conf)

Now that we have the config all done, let's save it to `/etc/lircd.conf` restart lircd `service lirc restart` and start sending the commands to the AC unit.

### Sending the commands from commandline

Sending the commands over command line is pretty straight forward:
`irsend -d /dev/lircd SEND_ONCE PI HEAT_AUTO_23`

Where the `PI` is the name of the remote and the `HEAT_AUTO_23` is the message we recorded earlier, and the `SEND_ONCE` just tells that we intend to send this message just once.

Now with the led properly pointing to the AC unit, sending the command should set the AC units settings as intended by the original message with these settings recorded.

So basically we are just replaying the original message.

#### Troubleshooting

* You can use your mobile phones camera to see if the IR led is blinking
* Be aware that the IR sensors are rather strict on where the signal can be seen, hard angles may cause it to fail to detect the command sent.

# TODO

* Create and document use of the web UI
* Add pics
* Add wiring diagrams

