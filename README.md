# si7021d

A Linux daemon that reads and returns data from a Si-7021 sensor. Written on Python.

It starts TCP server, accepts command `read` and returns JSON with sensor readings. 
You can send as many `read` commands during one connection as you want.

## License

MIT