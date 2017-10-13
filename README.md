# Byte Buffer
An implementation of a byte buffer-ish data structure in C that maintains read
and write positions, with support for common uses cases built in (serializing integers, 
reading/write to and from file descriptors).

## Usage


### Initialisation

As well as a capacity argument, the initialisation functions take a _resizable_ flag,
indicating whether or not the buffer should grow if necessary to accomodate a write operation:

````c
#include "byte_buffer.h"

...

uint64_t capacity = 8096;
bool resizable = true;
byte_buffer buf;
bb_init(&buf, capacity, resizable);
// or allocate and initialise a buffer
byte_buffer *buf_ptr = bb_alloc(capacity, resizable);
````
In addition to the standard initialisation functions, there are several more specialised ways to initialise a buffer:
````c
// create a complete clone, including the underlying array, of an existing buffer
byte_buffer buf_clone;
bb_clone(&buf_clone, &buf);
byte_buffer *buf_clone_ptr = bb_clone_alloc(&buf);

// create a shallow copy of a byte buffer without allocating a new underlying buffer
// should be used with care, but useful if you need read-only access to a buffer 
// eg. sending the same data out to multiple file descriptors/socketsin multiple contexts
byte_buffer buf_copy;
bb_copy(&buf_copy, &buf);
byte_buffer *buf_copy_ptr = bb_copy_alloc(&buf);
````
### Reading and Writing
There are four main pairs of read and write functions to handle arbitrary bytes, unsigned 64 bit integers, file descriptors and 'virtual' operations which are explained below. The first two are relatively straight forward, with the read and write functions both return an `int`, 0 on success, and -1 on failure. For writing failure means a lack of capacity to fulfull the write operation, and a failure to resize the buffer to accomodate. A read call fails without altering anything if the requested read size exceeds the amount of data available for reading from the specified buffer.
````c
uint8_t data[] = "this is some data";
uint64_t data_size = sizeof data;

bb_write_u64(&buf, data_size);
bb_write(&buf, data, data_size);

uint64_t read_size;
bb_read_u64(&read_size, &buf);

uint8_t read_buf[read_size]
bb_read(&read_buf, buf, read_size);

// we can also transfer directly between two buffers
bb_to_bb(&read_buf, &write_buf, count);
````
The file description functions work differently, instead returning `ssize_t`, with the value being the return value of the corresponding `read` or `write` call made. These functions do not take a `size` argument, instead they try to read or write as much as there is available in the buffer.
````c
int socket = server_connect();
fill_server_request(&write_buf);

// Write the request data from the buffer out to the socket
bb_read_to_fd(&write_buf, socket);

..

// Read in the response
bb_write_from_fd(&read_buf, socket);
````
### Compact, Reset and Clear

Writing to a buffer reduces the remaining write capacity, and increases the
amount that can be read from the buffer. Reading from the buffer does not 'consume'
the written data however, so a buffer will inevitably run out of room and either expand
or start failing to write eventually. To prevent this, a buffer can be _compacted_, which will copy any unread data to the beginning of
the buffer, and align the read and write positions accordingly.
````c
// fill up buf
bb_init(&buf, 1024, false);
bb_write(&buf, data, 1024);

..

u8 small_buf[512];
bb_read(small_buf, &buf, sizeof smallbuf);
// 512 bytes left still to read from buf, but 0 write capacity
bb_compact(&buf);
// buf now has 512 bytes to read from the beginning of the buffer and 512 bytes of write capacity

// Reset the buffer to its initial state. This does not alter the actual array contents, just positional info
bb_reset(&buf);
// If desired, the arrays contents can also be zeroed out
bb_reset_zero(&buf);

// Clear the buffer state. If the underlying array was allocated by an initialisation
// function, it will also be freed.
bb_clear(&buf);
// In addition to clearing a buffer's state, free the structure itself
bb_free(buf_ptr)
````





