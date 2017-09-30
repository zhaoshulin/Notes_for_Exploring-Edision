# Meet Edison
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

# First Contack
- Goal of mraa library: allows you to access the GPIO no matter which of the breakout boards you are using. (Ardunio, Galileo, RP3, etc.)
- Mraa uses a standard set of pin numbers which are mapped to appropriate pins on different devices. For example, mraa pin number 13 is mapped to on the Arduino breakout board pin 13, and to pin J17-14 on the mini breakout board.

# In C
- Why using C:
	- C is fast, obviously.
	- MCU can only be programmed in C.
> For many applications you have to be able to create a pulse of a known duration and within a specific time slot. The standard approach to Edison programming, i.e. using the Atom processor, is not good enough for this task because Linux is not a real time OS and this introduces uncertainties into pulse timing. So you have to move to programming the second CPU: the MCU, which can only be programmed in C now.

# Mraa GPIO
- 3 different pin numberings:
	- Arduino: standard Arduino shield pins. [Can be useful when you try to convert an Arduino sketch into C]
	- SYSFS: the raw/native Edison GOPI numbering, you can consider these numbers to be reality/physical. All GPIO lines are provided to the user as a file system with each pin corresponding to an I/O stream.
	- mraa: to make numbering independent of device and breakout board.
- I/O pins have 2 modes:
	- Pinmode 0: all lines are configured as digital I/O lines (i.e. pure GPIO lines)
	- Pinmode 1: some of the pins have a special functions like PWM or I2C.
> Default is Pinmode 0.
- Drive Characteristics:
	- GPIO Input:
		- 100 ns for 50 MHz clock when SoC is in S0 State.
		- 260 ns for 19.2 MHz clock when SoC is in S0i1 or S0i2 State.
		- 155.5 us for 32MHz clock (RTC) when SoC is in S0i3 State.
	- GPIO Output:
		- Each GPIO line can supply or sink 3mA.
		- Note: it does take quite a long time (10us) to access and change the state of an output line. Reason: mraa has to use SYSFS, which is slow.
> When you call `usleep(100)` you yield the thread to OS which might well schedule another thread to run, so you usually get a delay > 100us.
- 2 Methods of reading an input pin:
	- Polling: [faster]
		- Your thread is tied up doing nothing but polling.
		- However, OS still may suspend your thread, run something else and then restart your thread.
		- So, even with polling, you still cannot guarantee to respond to a change in an input line within a given time!
	- Interrupts: [slower]
		- your interrupt handler looks like this: `mraa_gpio_isr(pin, reason_causes_interrupt, pointer_to_function_triggered, pointer_to_args)`
		- In general, it is preferable to use interrupt. Unless you have to work with very fast changes in the input.
		- Edison provides a per pin interrupt system.
		- The pin interrupts are NOT true interrupts but software simulated interrupts!!!
			- When you set an interrupt on a pin, mraa creates a new thread and starts to make a blocking call to Linux system call `poll`.
			- Only returns when there is an event in the SYSFS file system for the file corresponding to the pin you have attached the interrupt handler to.
			- It is much slower than a hardware interrupt handling.
			- Because the polling loop is running on new thread and is scheduled by Linux in the usual way, it could take milliseconds int the worst case to respond to a change.
			- Interrupt is disabled when handler is executing. You may lose events.
			- Mraa will only spin up a single thread for each pin. So you can only have associate one interrupt handler per pin. For example, you will not be able to set up a chain of handlers that deal with a rising edge, then a falling edge.
> Note: Right now, if without doing optimizations, Edison can only be millisecond's fast, but not macrosecond's fast!

> Next Chapter will try to solve this timing problem...

# Fast Memory Mapped IO
- Fast memory mapped mode allows Edison to generate pulses as short as 0.25 microseconds wide and to work with input pulses in the 10 microseconds region. [on Atom CPU]
- Motivations to be faster:
	- bit-banging: e.g., sensors using one wire protocols
	- Linux is not a real time OS, but time slides sharing. Your program could be suspended at any time and it could be suspended for milliseconds. If your program is performing a big-banging operation the entire exchange could be brought to a halt by another process that is given the CPU for its time slice. 
		- This could cause the protocol to be broken and have to start over and hope that the transaction could be complete this time.
	- In a word, under a standard Linux OS, you can not do real-time and big-banging, because your process can be suspended any time!
- `usleep(10)` will yield the thread to OS and this incurs an additional 50 microsecond penalty due to calling the scheduler, totally 78 microsecond fixed overhead.
> `usleep(10)` only promises that your thread will not restart for AT LEAST 10 microseconds.
- Busy waiting doesn't yield CPU. But OS might preempt your thread. So, no guaranty.
- Why directly GPIO access is so slow:
	- mraa is using the Linux SYSFS subsystem:
		- Good: making the GPIO available to almost anything that runs under Linux
		- Bad: high overhead!
- Solution: allow the software to write directly to the memory locations where the GPIO port registers live. [0.3 microsecond, 60X faster!]
> Note: memory mapped IO only substitutes read() and write() for a given pin, all other operations such as setting the lines' direction, will still use a SYSFS call.
- SYSFS can produce a 0.03MHz pulse train; while memory mapping can produce a 2 MHz pulse train!
- In Edison, you can simulate time with a for loop: `time = 0.66*i + 0.548 (microseconds)`. It is much better than using a system call of `clock_gettime()`
- Using fast memory mapped IO, you can work with signals in from 0.25 microseconds output, and with about 5 microseconds inputs.
- How to use fast memory mapped mode for pin 13:
			
			mraa_gpio_context pin = mraa_gpio_init(13);
			mraa_gpio_use_mmaped(pin, 1);

# Near Realtime Linux
- In Quark MCU of Edison, there is no OS involved. Which means, on OS scheduler and you are in complete charge of the CPU.
- However, In Edison Atom, scheduling is a problem when you are trying to write a real time program.
> A thread can be suspended before its time slice is up because it has to wait for IO or because it is blocked in some other way.

> By default all of the standard threads have `0` priority and are scheduled by the normal scheduler.
> So, if you start a FIFO scheduled thread with priority 1, it will start immediately on one of the two cores available.
> If the process never makes a call that causes it to wait for IO say or become blocked in some other way then it will execute without being interrupted by any other processes.

- 5 supported scheduling:
	- `SCHED_OTHER`: the standard round-robin time-sharing policy
	- `SCHED_BATCH`: for batching style execution of processes
	- `SCHED_IDLE`: for running very low priority background jobs
	- `SCHED_FIFO`: real time, FIFO
	- `SCHED_RR`: real time, RR
- How to set your thread's scheduling:
		#include <sched.h>

		const struct sched_param priority={1};
		sched_setschduler(0, SCHED_FIFO, &priority);

- After using `SCHED_FIFO`, your program will have very less interrupts and independent of CPU loads. 
- However, there would be several problems by doing this:
	- About every 10 minutes, the wifi link will fail!!!
		- Because: the wifi thread doesn't get to run sufficiently often.
		- So, you have to yield at regular intervals!
	- Every ICH chip (including Atom), there is a System Management Interrupt(SMI). An SMI is sth that happens outside of the OS and it is often necessary for the correct operation of the hardware. 
	> The bottom line is that SMIs cannot easily be turned off and this problem that effects all OS, including real time OS.
- This motives us to:
	- Use MCU, or additional hardware to carry out the operation away from the software like the UART or the PWM GPIO lines

# Sophisticated GPIO-Pulse Width Modulation
- The goal of this chapter: move the not-fast-enough problem away from the Atom CPU.
- There are 4 PWM GPIO output pins, which can constituently generate 5X faster PWM than memory mapped pins once setting is done, because of no CPU involved.
> This is probably useless for `sensor.read()`......

# Sophisticated GPIO-I2C
- I2C is a simple bus that uses 2 active wires:
	- SDA: for data
	- SCL: for clock, set the speed of data transfer
- Edison sends a request of reading a I2C sensor:
	- Sensor sends back `ACK` if ready
	- Sensor sends back `NAK` if not ready

# I2C-Measuring Temperature
- 2 options when sensor is not ready:
	- Hold Master: Edison is held waiting for the data
	- No Hold Master: Edison is released to do something else and poll for the data until it is ready

# Life at 1.8V
- Edison SoC works with 1.8V logic.
> This chapter is useless...

# The DS18B20 1-Wire Temperature Sensor
- Edison doesn't support 1-Wire Protocol. Because: fast switching between input and output.

# Using the SPI Bus
- SPI is used for high data rate streaming, e.g., video data, LCD displays, ADC.
- In principle, Edison only supports at most 2 SPI devices.
- When you keep reading, the first read takes longer than subsequent reads, because:
	- The pin is setup at this first point.