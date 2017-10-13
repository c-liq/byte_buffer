# Byte Buffer
An implementation of a byte buffer-ish data structure in C that maintains read
and write positions, with support for common uses cases built in (serializing integers, 
reading/write to and from file descriptors).

##Usage


###Initialisation

As well as a capacity argument, the initialisation functions take a _resizable_ flag,
indicating whether or not the buffer should grow if necessary to accomodate a write operation:

````
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
````
// create a complete clone, including the underlying array, of an existing buffer
byte_buffer buf_clone;
bb_clone(&buf_clone, &buf);
byte_buffer *buf_clone_ptr = bb_clone_alloc(buf_clone_ptr, &buf);

// create a shallow copy of a byte buffer without allocating a new underlying buffer
// should be used with care, but useful if you need read-only access to a buffer 
// eg. sending the same data out to multiple file descriptors/socketsin multiple contexts
byte_buffer buf_copy;
bb_copy(&buf_copy, &buf);
byte_buffer *buf_copy_ptr = bb_copy_alloc(buf_copy_ptr, &buf);
````
###Reading and Writing
There are three main pairs of functions to read and write from/to a buffer. _bb_read_ and _bb_write_ access an 
abritrary number of bytes in an array. Additionally there are functions with _u64 suffixes for unsigned 64 bit integers, 
and _fd to read and write from file descriptors. All read and write functions return 0 on success, and -1 on failure. In the case of writing, failure
means there was insufficient space to complete the write, and either the resize operation failed or the buffer is flagged
as unresizable.
````
uint8_t data[] = "this is a message";
uint64_t data_size = sizeof data;

bb_write_u64(&buf, data_size);
bb_write(&buf, data, data_size);

uint64_t read_size;
bb_read_u64(&read_size, &buf);

uint8_t read_buf[read_size]
bb_read(&read_buf, buf, read_size);
````
### Resetting, compacting and cleaning up

Writing to a buffer reduces the remaining write capacity, and increases the
amount that can be read from the buffer. Reading from the buffer does not 'consume'
the written data however, so a buffer will inevitably run out of room and either expand
or start failing to write eventually. To prevent this, a buffer can be _compacted_, which will copy any unread data to the beginning of
the buffer, and align the read and write positions accordingly.
````
// fill up buf
bb_init(&buf, 1024, false);
bb_write(&buf, data, 1024);

..

u8 small_buf[512];
bb_read(small_buf, &buf, sizeof smallbuf);

// 512 bytes left still to read from buf, but 0 write capacity
bb_compact(&buf);
// buf now has 512 bytes to read from the beginning of the buffer
// and 512 bytes of write space

// we can also reset a buffer to its initial state. this only restores the read and write
// state, it does not clear or alter the buffer at all. Calling compact on a buffer
// with no data available to read is equivalent to resetting it.
bb_reset(&buf);
// If desired, the arrays contents can also be zeroed out
bb_reset_zero(&buf);

// Clear a buffer's state. If the underlying data array was initialised by a bb_init/alloc
// call, then this will also be freed
bb_clear(&buf);
// If the buffer structure itself was created by a _alloc call, it can be freed
// at the same time as the structure is cleared
bb_free(&buf)
````





