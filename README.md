# the ESP8266

From the homepage of the [vendor](https://espressif.com/en/products/esp8266/):

* ESP8266 is a highly integrated chip designed for the needs of a new connected world. It offers a complete and self-contained WiFi networking solution, allowing it to either host the application or to offload all WiFi networking functions from another application processor.*

Condensed specs:

* WiFi (802.11 b/g/n, direct connect and AP mode, WEP, WPA/WPA2)
* integrated TCP/IP protocol stack
* uProcessor (Xtensa LX106) @ 80Mhz, 96kB data RAM, 64kB instruction RAM
* SDIO 1.1/2.0, SPI, UART, GPIO
* Vcc 3.3V, I = 0.5uA..215mA
* application interface uses AT commands

# esp-open-sdk

Compiled by [pfalcon](https://github.com/pfalcon) using

* Xtensa lx106 architecture tool-chain
* ESP8266 IoT SDK from Espressif Systems

provided via [GitHub](https://github.com/pfalcon/esp-open-sdk).

It contains everything to build and deploy firmware images for the ESP8266.

You should check the IoT SDK API documentation - see the readme.txt in folder '/home/espbuilder/esp-open-sdk/esp_iot_sdk_v1.0.0/document/English/'. The current version of the ESP8266 SDK API GUI can be found [here](http://bbs.espressif.com/download/file.php?id=1160).

# usage

## hardware

The basic steps to upload code to the ESP8266 can be found [here](https://github.com/esp8266/esp8266-wiki/wiki/Uploading).
Just connect the ESP8266 module using a 3.3V-RS232-USB-adapter (eg. FTDI232 adapter) and a 3.3V power supply (I started 
using the 3.3V rail from the FTDI232 - but this is not able to deliver enough power and the ESP8266 acts wired...):

```
ESP8266               PWR SPLY
------+              +---------
   Vcc|-+------------| 3.3V / 300mA
      | |            |
CHP_EN|-+         +--| GND  
      | |         |  |
   RST|-+         |  +---------
      |           |   
 GPIO2|*          |  
      |     *     | 
 GPIO0|-----+     | 
      |     |     |  RS232-USB
      |      \SW0 |  +---------
      |     |     |  |
   GND|-----+-----+--|GND
      |              |
    RX|--------------|TX
      |              |
    TX|--------------|RX
------+              +---------

* GPIO0 and GPIO2 are connected to an internal pull up resistor

pinout ESP-01, ver. 02, top view

--------------+
     RX  O  O | Vcc
  GPIO0  O  O | RST
  GPIO2  O  O | CHP_EN
    GND  O  O | TX
--------------+
```

The default baud rate during boot is 115200 (8N1). After boot my modules switched down to 9600 (if nothing works, check 57600 and 115200). If your module reboots (eg. caused by the watchdog) the firmware spills out the reason of the reboot @ 115200. So it is
a good idea to switch also on the application level to 115200 for being able to read the boot messages without the need to
reconfigure the terminal for interacting with the user-application...

Check if the setup is working by running the *AT* command:

```
host>sudo screen /dev/ttyUSB0 9600
AT

OK
AT+RST

OK
<some strange stuff>
[System Ready, Vendor:www.ai-thinker.com]
```

**NOTE:** Newer modules (firmware >= 0.92) expect CR/LF ('\r\n') for command/line termination. First press
the ENTER key (generates 0x0d - \r) and then CTRL+j (generates 0x0a - \n).

To enable the *download boot mode* simply enable switch SW0 (pull GPIO0 down) during power on / after hardware reset.

**NOTE:** Software reset (AT+RST) won't work for switching into *download boot mode*.

## software

### blinky example

Create the image:

```
host> docker build -t hecke/esp-open-sdk:1.5.2 .
```

Start the container:

```
host> docker run -ti --privileged hecke/esp-open-sdk:1.5.2
```

Build and flash the blinky example:

```
espbuilder@container> cd source-code-examples/blinky/
espbuilder@container> make
CC user/user_main.c
AR build/app_app.a
LD build/app.out
FW firmware/0x00000.bin
FW firmware/0x40000.bin
espbuilder@container> make flash
esptool.py --port /dev/ttyUSB0 write_flash 0x00000 firmware/0x00000.bin 0x40000 firmware/0x40000.bin
Connecting...
Erasing flash...
Writing at 0x00007400... (100 %)
Erasing flash...
Writing at 0x00065400... (100 %)

Leaving...
espbuilder@container>
```

Reset/re-power the ESP8266 module and check if GPIO2 toggling with f=0.5 Hz.

**NOTE:** The default port used to flash the firmware is /dev/ttyUSB0. To override the default, 
use the attribute 'ESPPORT', eg:

```
...
espbuilder@container> make ESPPORT=/dev/ttyUSB42 flash
...
```

### minimal project

Create a project folder on the host machine:

```
host> mkdir -p /home/hecke/projects/esp8266-minimal
```

Start the container using the project folder as a shared directory:

```
host> docker run -ti -v /home/hecke/projects/esp8266-minimal:/home/espbuilder/esp8266-minimal --privileged hecke/esp-open-sdk:1.5.2
```

Setup project environment (copy basic Makefile from blinky, create source folder):

```
espbuilder@container> cd esp8266-minimal
espbuilder@container> cp source-code-examples/blinky/Makefile .
espbuilder@container> mkdir user
espbuilder@container> touch user/user_config.h
```

Just c&p the following lines into user/main.c. You can do this on your host machine or within the container.

```
#include "ets_sys.h"
#include "osapi.h"
#include "gpio.h"
#include "os_type.h"

#define user_procTaskPrio 0
#define user_procTaskQueueLen 1
os_event_t user_procTaskQueue[user_procTaskQueueLen];
static void user_procTask(os_event_t *events);

//NOP - just trigger myself
static void ICACHE_FLASH_ATTR user_procTask(os_event_t *events)
{
	os_delay_us(10);

  system_os_post(user_procTaskPrio, 0, 0 );
}

void ICACHE_FLASH_ATTR user_init()
{
	system_os_task(user_procTask, user_procTaskPrio,user_procTaskQueue, user_procTaskQueueLen);

  system_os_post(user_procTaskPrio, 0, 0 );
}
```

To build the minimal example just run make:

```
espbuilder@container> make
CC user/main.c
AR build/app_app.a
LD build/app.out
FW firmware/0x00000.bin
FW firmware/0x40000.bin
...
```

# links

* [esp8266-wiki](https://github.com/esp8266/esp8266-wiki/wiki)
 * [AT command set](https://github.com/esp8266/esp8266-wiki/wiki/at_0.9.1)
 * [memory map](https://github.com/esp8266/esp8266-wiki/wiki/Memory-Map)
 * [boot process](https://github.com/esp8266/esp8266-wiki/wiki/Boot-Process)
* [ESP8266 Community Forum](http://www.esp8266.com)
* blog entries on [hack-a-day](http://hackaday.com/?s=esp8266)
 * awesome [color TV broadcast](http://hackaday.com/2016/03/01/color-tv-broadcasts-are-esp8266s-newest-trick/) using ESP8266 by cnlohr
* [esp8266 GPIO performance](http://naberius.de/2015/05/14/esp8266-gpio-output-performance/)

There are other ESP8266 tool-chains (like [this](https://github.com/cnlohr/ws2812esp8266) one) and a [Arduino compatible IDE](https://github.com/esp8266/arduino) that supports the ESP8266.

