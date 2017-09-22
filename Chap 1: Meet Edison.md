- Edison has 2 separate processors:
	1. Atom Host CPU (500 MHz): running Linux OS
	2. Quark processor (100 MHz) acting as an MCU Micro-controller: running RTOS derived OS, do nothing but run the program you downloaded.
![Host CPU vs. MCU](http://www.i-programmer.info/images/stories/News/2015/May/B/edisonmcu.jpg)
- Which board for which applications:
	- Arduino Uno is good for projects that involve a lot of digital and analog I/O and minimal communication (because of no network functions) and where size and power requirements are not an issue. It is not low power as standard but there are variants suitable for special purposes such as wearables.
	- Raspberry Pi is a full computer with good communications. Use it when you need to do complicated things.
	- Edison is also a full computer but without video and keyboard interfaces. It has wifi and BT as standard and so is good at communication. Its low power makes it suitable for battery operation.
- The native logic level for the Edison is 1.8V and its current drive is just 3mA. In other words, there are not TTL or CMOS levels of the sort you encounter in the Arduino and the Raspberry Pi.
- Arduino breakout board not only gives you Arduino pin outs, but it level shifts the 1.8V logic of the Edison to the 5V logic of the Arduino. You can also use full Arduino IDE to develop software. However, it is not the lightest weight option for working with the Edison but it is very capable.
- The native mini-breakout board is harder to work with but it is the one you need to master to get the real Edison experience.