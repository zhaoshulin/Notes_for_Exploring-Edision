- I2C is a simple bus that uses 2 active wires:
	- SDA: for data
	- SCL: for clock, set the speed of data transfer
- Edison sends a request of reading a I2C sensor:
	- Sensor sends back `ACK` if ready
	- Sensor sends back `NAK` if not ready