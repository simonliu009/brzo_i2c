Brzo I2C
===================
Brzo i2c implements an i2c master and is written in assembly language for the esp8266. The core features are:

 - Fast mode plus speeds 
	 - 800 KHz @ 80 MHz CPU frequency
	 - 1 MHz @ 160 MHz CPU frequency
 - Precise timing for SCL and SDA, including removal of spikes
 - i2c transactions: Group any combination of i2c writes or reads together, either with or without repeated start.

For further information you may refer to the [wiki](https://github.com/pasko-zh/brzo_i2c/wiki)

HW and Tool Chain Support
-------

 - Brzo i2c supports **only** esp8266 modules because it is implented in Xtensa assembly language.
 - The code is tested for the Arduino tool chain only (however, it should work with the native tool chain, too). Tested versions:
	 - Arduino IDE 1.6.5
	 - ESP8266 Arduino Core 2.1.0

Brzo i2c was tested with several i2c devices. If you find one which is not working, please open an issue.

How to use the Brzo I2C Library
===============================
Since it supports the esp8266 only, you should put the library into the corresponding esp8266 arduino core directory, for instance under Windows: `...\packages\esp8266\hardware\esp8266\2.1.0\libraries` 

Include brzo_i2c in your sketch as any other library with `#include "brzo_i2c.h"`. 

I2C Setup
----------------

To setup your i2c bus you need to call at least once `brzo_i2c_setup`.  

`brzo_i2c_setup(uint8_t sda, uint8_t scl, uint32_t clock_stretch_time_out_usec)`.  
Input

 - The number of your SDA Pin
 - The number of your SCL Pin
 - The timeout for a slave's clock stretch in micro seconds

Returns

 - None
 
Example:
`brzo_i2c_setup(5, 12, 200);`

I2C Transactions
----------------

The motivation for i2c transactions is found on the [wiki](https://github.com/pasko-zh/brzo_i2c/wiki#i2c-transactions). An i2c transactions always begins with `brzo_i2c_start_transaction` and ends with `brzo_i2c_end_transaction`.  Inside a transaction you can combine any number i2c commands with or without a repeated start. A transaction is bound to one i2c slave and all commands are executed at the same SCL speed. Thus, brzo_i2c_start_transaction takes the slave address and SCL speed in KHz as input parameters.

**Begin a transaction**

`brzo_i2c_start_transaction(uint8_t slave_address, uint16_t SCL_frequency_KHz)`

Input

 - I2c slave address (7 bit encoded)
 - SCL speed to be used for the entire transaction in KHz (SCL speed is rounded to the next 100-level, e.g., 651 will be rounded up to 700)


Returns

 - None

Example: Start a transaction to an i2c slave at 0x20 with 1 MHz SCL speed.
`brzo_i2c_start_transaction(0x20, 1000);`


**I2C write**

`brzo_i2c_write(uint8_t *data, uint8_t no_of_bytes, boolean repeated_start)`

> **Note:** 
> - You can only call `brzo_i2c_write` inside a transaction!
> - Of course, there is error checking inside the write command. However, errors are only reported at the end of a transaction. 


Input
 - A pointer to an array of bytes: Either use the base address, e.g., `brzo_i2c_write(buffer, 2, ..)` or use the address of a specific array element, e.g. `brzo_i2c_write(&buffer[4], 2, ..);`. The former writes `buffer[0]` and `buffer[1]`, the latter  `buffer[4]` and `buffer[5]`
 - The number of bytes to be written
 - Repeated start if needed
 
Returns
- None

Example
- From (the previously defined)  `uint8_t buffer[4]` write 2 bytes (`buffer[0]`, `buffer[1]`) and finish the command with a STOP,  i.e. no repeated start.
`brzo_i2c_write(buffer, 2, false);`

**I2C Read**

`brzo_i2c_read(uint8_t *data, uint8_t no_of_bytes, boolean repeated_start)`

> **Note:** 
> - You can only call `brzo_i2c_read` inside a transaction!
> - Of course, there is error checking inside the read command. However, errors are only reported at the end of a transaction.


Input
 - A pointer to an array of bytes: Either use the base address, e.g., `brzo_i2c_read(buffer, 2, ..)` or use the address of a specific array element, e.g. `brzo_i2c_read(&buffer[4], 2, ..);`. The former receives two bytes and saves them to `buffer[0]` and `buffer[1]`, the latter to `buffer[4]` and `buffer[5]`
 - The number of bytes to be read
 - Repeated start if needed
 
Returns
- None

Example
- Into (the previously defined) `uint8_t buffer[4]` store 3 bytes (`buffer[0]`, `buffer[1]`, `buffer[2]`) and finish the read with a repeated start, i.e. do not send a STOP.
`brzo_i2c_read(buffer, 3, true);`

**End a Transaction**

`uint8_t brzo_i2c_end_transaction()`

> **Note:** 
> - The last i2c command before calling `brzo_i2c_end_transaction()` must not contain a repeated start, therefore use `brzo_i2c_read/write(.., .., false)` to send a STOP sequence instead of another repeated START.
> - Refer to the [i2c specification](http://www.i2c-bus.org/repeated-start-condition) about repeated starts and how to "end" them


Input
 - None
 
Returns
- 0 : All i2c commands were executed without errors
- errors
	 - 1 : Bus not free
	 - 2 : Not ACK ("NACK") by slave during write: Either the slave did not respond to the given slave address; or the slave did not ACK a byte transferred by the master.
	 - 4 : Not ACK ("NACK") by slave during read, i.e. slave did not respond to the given slave address
	 - 8 : Clock Stretching by slave exceeded maximum clock stretching time. Most probably, there is a bus stall now!
	 - 16: Read was called with 0 bytes to be read by the master. Command not sent to the slave, since this could yield to a bus stall


Example of I2C Transactions
----------------
Examples of sketches are located in [/examples](https://github.com/pasko-zh/brzo_i2c/tree/master/examples) folder.

```c
#include "brzo_i2c.h"

uint8_t SDA_PIN = 5;
uint8_t SCL_PIN = 12;
uint8_t ADT7420_ADR = 0x49;
uint8_t buffer[3];
uint8_t return_code = 0;
uint32_t ADC_code = 0;
long temp;

void setup() {
  // Setup i2c with clock stretching timeout of 2000 usec
  brzo_i2c_setup(SDA_PIN, SCL_PIN, 2000);
...
}

...

// Start a transaction to the ADT7420 at 400 KHz

brzo_i2c_start_transaction(ADT7420_ADR, 400);
  buffer[0] = 0x03; // select configuration register
  buffer[1] = 0xA0; // select 16 bit resolution
  brzo_i2c_write(buffer, 2, false); // Write two bytes with no repeated start
  buffer[0] = 0x00; // select temperature register
  brzo_i2c_write(buffer, 1, true); // Write one byte WITH repeated start
  brzo_i2c_read(buffer, 2, false); // Read temperature, two bytes
return_code = brzo_i2c_end_transaction();

if (return_code == 0) {
  ADC_code = ((buffer[0] << 8) | buffer[1]);
  temp = ADC_code / 128.0;
 }
else {
  // Error Handling here...
}
```


