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