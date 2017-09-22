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