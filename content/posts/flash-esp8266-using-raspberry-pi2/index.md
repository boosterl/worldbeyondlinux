---
title: "Flash an ESP8266 Using a Raspberry Pi"
date: 2024-12-23T17:50:44+01:00
draft: false
description: "In this guide I will be explaining how I flash an ESP8266 chip using a Raspberry Pi. More specifically; a Raspberry Pi 2. But any model with GPIO pins will do. In this guide I will be flashing MicroPython on the chip."
---
In this guide I will be explaining how I flash an ESP8266 chip using a
Raspberry Pi. More specifically; a Raspberry Pi 2. But any model with GPIO pins
will do. In this guide I will be flashing [MicroPython](https://micropython.org/)
on the chip.

First of all an important clarification; when I'm talking about an ESP8266, I
actually mean an ESP12F mounted on a development board with no USB-port or
buttons to go into flash-mode. The difference in versions of an ESP8266 was not
that clear to me in the beginning. But now it's a bit more clear to me.
Basically it comes down to this; ESP8266 is the actual SoC, this is then mounted
on a module, which contains an antenna etc. Example of these modules are; ESP12F
and ESP12E. This combination can then be used on a development board. This
development board can contain breadboard pins, a USB connector, buttons to go
into flash mode etc. My module only contains breadboard pins. This is why I was
struggling in the beginning of using this chip to actually being able to flash
something on it. But now I know how I can flash something on the chip only using
some basic electronic components and a Raspberry Pi I had lying around.

The electronic parts needed for this are the following:
* Two breadboards (the ESP8266 module is too big to fit onto one, and still have
pins reachable)
* Four 10k ohm resistors
* Two buttons
* Some jumper wires
* A Raspberry Pi
* An AMS1117 voltage divider

The setup should look like this:
![Wiring](/posts/flash-esp8266-using-raspberry-pi2/images/flashing_scheme.svg)

What we are basically doing here is powering the board, and making sure the
correct pins are in the correct state (pulled down or up) to be in running mode.
The AMS1117 voltage divider is used so that we can draw the current from the 5v
rail of the Raspberry Pi and convert this into the 3.3v the ESP8266 needs. As
far as I know the 3.3v rails on the Raspberry Pi can only generate a small
amount of current, probably not enough to power an ESP8266. The buttons are
used to put the chip into download-mode. But when they are not pressed, the
chip is in running mode. Here you can see a table showing how the pins need to
be configured depending on the mode:
| Mode | CH_PD(EN) | RST | GPIO15 | GPIO0 | GPIO2 | TXD0 |
|---|---|---|---|---|---|---|
| Download mode| high | high | low | low | high | high |
| Running mode| high | high | low | high | high | high |

For the software, I used Raspberry Pi OS. The serial hardware will need to be
enabled. I go more into detail about this in [this](/posts/how-to-use-a-pi-instead-of-a-usb-console-cable)
article. A short overview of what needs to be done:
1. Start raspi-config: `sudo raspi-config`
2. Select option 3 - Interface Options.
3. Select option P6 - Serial Port.
4. At the prompt `Would you like a login shell to be accessible over serial?` answer 'No'
5. At the prompt `Would you like the serial port hardware to be enabled? answer` 'Yes'
6. Exit raspi-config and reboot.

Only needed when you have a Pi with built-in Bluetooth:
1. Add `dtoverlay=disable-bt` to the `/boot/config.txt`-file.
2. Execute `sudo systemctl disable hciuart`.
3. Reboot

We also need some utilities, namely [Picocom](https://github.com/npat-efault/picocom)
for opening a serial connection, and [esptool](https://github.com/espressif/esptool)
for actually flashing stuff to the ESP8266. Both these utilities can be installed
by executing the following commands:
```console
$ sudo apt update
$ sudo apt install picocom esptool
```

If everything is physically connected, and software is installed, you could
already try to open a serial connection:
```
$ picocom -b 115200 /dev/ttyAMA0
```
If you have not previously put flashed anything to the ESP8266, you probably
have the default espressif ROM on the chip. You could try the `AT`-command:
```console
> AT
```

Now for the exciting bit; going into download mode! To go into download-mode,
you will need to hold down the button going to GPIO0. While holding that button
down, press the button going towards the RST pin (this resets the board). You
should now be in download-mode, and you can let go of the button going towards
GPIO0.

To verify you are actually in download mode, you could execute the following
command on the Raspberry Pi:
```console
$ esptool --port /dev/ttyAMA0 flash_id
```

This should return something like this:
```
esptool.py v2.5.1                         
Serial port /dev/ttyAMA0
Connecting....                    
Detecting chip type... ESP8266        
Chip is ESP8266EX                                
Features: WiFi                                                                                            
MAC: ...
Enabling default SPI flash mode...                                                                        
Manufacturer: e0                   
Device: 4016
Detected flash size: 4MB
Hard resetting via RTS pin...
```

For actually flashing something to it, I will be flashing the latest version
(as of writing this) of [MicroPython](https://micropython.org/) to the board.
You can download the latest version from [here](https://micropython.org/download/ESP8266_GENERIC/).
Once the file is downloaded, and available in the current working directory,
the following commands can be executed:
```console
$ esptool --port /dev/ttyAMA0 erase_flash
$ esptool --port /dev/ttyAMA0 --baud 460800 write_flash --flash_size=detect 0 ESP8266_GENERIC-20241129-v1.24.1.bin 
```

The `erase_flash` command actually failed on my board, but this didn't seem to
effect the final result.

That's it. Happy holidays!
