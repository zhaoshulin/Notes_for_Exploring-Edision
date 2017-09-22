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