# OttoLight

## Overview

OttoLight is a framework for networked communication between a 
web service and master controller to any number of nearby client 
controllers. It is currently designed for use with WS2812/WS2811 
LED strips, although the clients can preform any remote task.

## Details

This system is built to run on the Texas Instrument [MSP430]
(http://www.ti.com/ww/en/launchpad/launchpads-msp430.html#tabs)
series(Launchpad) microcontrollers with Anaren [AIR BoosterPack]
(https://www.anaren.com/air/air-boosterpack).

The one master controller communicates with a PC host over USB 
serial, where is receives commands to relay to the clients. The 
clients in turn receive and parse messages from the master and
act accordingly.

Each client is connected to an LED strip(equivalent to 
Adafruit's [Neopixel](http://www.adafruit.com/category/168) and
listens for commands during the delay between output frames. 
Because RF communication and pushing a new frame are each 100%
blocking tasks, it is possible for the client to miss a message 
while processing LED information. Therefore, the master controller
repeats its transmission until an echo is received from the client.

## Background

I wrote this system at [Hack_UTD](http://hackutd.co/), The
University of Texas at Dallas's first major student hackathon.
I worked on a team by myself with occasional hardware help from 
the present TI engineers who graciously provided most of the 
hardware. Because I only had twenty-four hours to get a functional
prototype, the code may seem a bit rough. I'm planning on cleaning 
it up and documenting the codebase.

## Installation and Usage

This program has two main requirements, and a few other small 
dependencies.

* [Energia](http://energia.nu/) Launchpad Prototyping platform
  * [WS2811 Launchpad Driver](https://github.com/ILAMtitan/WS2811Driver)
* [Node.js](http://nodejs.org/)
  * [Express](https://github.com/voodootikigod/node-serialport)
  * [node-serialport](https://github.com/voodootikigod/node-serialport)

After installing those dependencies, download or clone this repository
and keep following the below instructions.

### Master

Open `./Master/Master.ino` in Energia and make sure that you have all of the
dependencies and the code compiles. Select the correct board and serial COM
port, then burn the program. Open the serial monitor and reset the board to
confirm that serial communication(9600 baud) works.

### Client

Open `./Client/Client.ino` in Energia and modify the leading lines to match 
your board and number of LEDs. You should assign each device a unique ID
starting at one(zero is reserved for the master) so that they don't 
crosstalk and interfere. Burn the script and by default, it should load
a fast white chaser.

Keep in mind that the LED Driver library takes 3 bytes of RAM per LED in
order to function. If you use more than the board has available, it will
silently fail after running out. In my experience, the MSP430G2 can't run
many more than 30 LED's and the MSP430F5529 can run at least 150.

### Webserver

In a command prompt, navigate to `./webserver` and execute `auto.js`, followed
by the name of the COM port of the master controller.

```
node auto.js COM6
```
  
In your web browser, navigate to http://localhost/ and you should see a simple
web interface. As you select options and click `Send`, traffic should be logged
in the Node.js terminal and the client LED's should follow.

The web interface is currently configured only three client devices, but it can 
easily be expanded.

![Default web interface](https://raw.githubusercontent.com/GGreenwood/OttoLight/master/interface.png)

## Communication
Currently, there are two communication formats: one for the master to send to clients, and one for the clients to use while echoing back. The content of the first byte(DESTINATION_ID) determines which message type is being used. The master has a device ID of 0, while the clients increment from 1 to 255. 

### Master to Client

```
{DESTINATION_ID, MODE, DELAY, RED, GREEN, BLUE}
```

This six-byte format is used only when the destination device ID is nonzero.

* DESTINATION_ID
 * The ID of the device receiving the message
* MODE
 * The ID of the animation to use. 0 is off, 1 is a color sweep, 2 is a chase, etc.
* DELAY
 * The number of milliseconds to delay in between frames. If a value lower than 15ms is received, it is rounded up to 15.
* RED
 * The RGB value in red to display.
* GREEN
 * The RGB value in green to display.
* BLUE
 * The RGB value in blue to display.

### Client to Master

```
{DESTINATION_ID, MODE, SENDER_ID, SENDER_TYPE}
```

This four-byte format is used for clients communicating back to the master. In this case, the destination device ID is always 0.

* DESTINATION_ID
 * The ID of the device receiving the message (zero)
* MODE
 * The purpose of the message. 0 is used to confirm a successful transmission
* SENDER_ID
 * The device ID of the transmitting device
* SENDER_TYPE
 * The type of device sending the message. Is intended for more general messaging, but is currently just 1 for LED strips.
