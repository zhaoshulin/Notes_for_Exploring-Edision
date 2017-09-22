- Why using C:
	- C is fast, obviously.
	- MCU can only be programmed in C.
> For many applications you have to be able to create a pulse of a known duration and within a specific time slot. The standard approach to Edison programming, i.e. using the Atom processor, is not good enough for this task because Linux is not a real time OS and this introduces uncertainties into pulse timing. So you have to move to programming the second CPU: the MCU, which can only be programmed in C now.