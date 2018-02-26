# Chapter 3: File I/O
These notes are taken with the assumption that you have read chapter 1 notes. Redundant information has been omitted in the interest of shorter notes (also I'm lazy).

## 3.1 - Introduction
- Most I/O on a UNIX system can be performed using only five functions: `open, read, write, lseek`, and `close`.
- These functions are referred to as **unbuffered I/O**, meaning that each `read` or `write` invokes a system call in the kernel.

## 3.2 - File Descriptors
- To the kernel, all open files are referred to by **file descriptors**, which is just a non-negative integer.
- We pass file descriptors to the unbuffered I/O functions.
- File descriptors range from 0 to `OPEN_MAX - 1`. 
    - *On newer systems, this limit is essentially infinite and only bounded by the amount of physical memory on the system.*

## 3.3 - `open` and `openat` Functions
- A file is opened or created by calling one of the following functions:
```c
#include <fcntl.h>

int open(const char *path, int oflag, ... /* mode_t mode */);

int openat(int fd, const char *path, int oflag, ... /* mode_t mode */);

/* Both return file descriptor on success, -1 on error. */
```

- The `oflag` argument is formed by ORing one or more constants from the `<fcntl.h>` header.
    - One or more of the following are required: `O_RDONLY, O_WRONLY, O_RDWR, O_EXEC, O_SEARCH`.
    - There are many more optional arguments.
- The returned file descriptor is guaranteed to be the lowest-numbered, unused file descriptor. You can use this in a clever way, such as closing standard error (fd=1) and opening a new file, which you know will have fd=1. However, there are better ways to do this (see section 3.12).
- If `openat` is given an absolute path, then it behaves like `open`.
- **TOCTTOU Error**: "Time-Of-Check-To-Time-Of-Use" error is that a program is vulnerable if it makes two file-based function call where the second call depends on the results of the first call (race condition).

### Filename and Pathname Truncation
- If `NAME_MAX` is 14 and you try to create a file with a filename containing 15 characters, old UNIX systems would truncate the name. BSD-Derived systems would report an error.
- Problem: You have no way to determine what original filename was as it might have been truncated.
- If the file system decides not to truncate, `errno` is set to `ENAMETOOLONG`. Modern filesystems support names at least 255 characters long, so this is usually not a problem.

## 3.4 - `creat` Function
Can create a new file with following code:
```c
#include <fcntl.h>

int creat(const char *path, mode_t mode);
/* Returns fd opened for write-only on success, -1 on error. */
```

This function is equivalent to:
```c
open(path, O_WRONLY | O_CREAT | O_TRUNC, mode);
```

*TBH don't use `creat`, use `open` since the file is opened only for writing, so you'd have to call `creat`, `close`, and then `open`.*

## 3.5 - `close` Function
Can close a file like so:
```c
#include <unistd.h>

int close(int fd);
/* Returns 0 on success, -1 on error. */
```

Closing a file also releases any locks that a process may have on a file (see multithreading concepts in 14.3).

Major benefit of using `close` vs `fclose` is that the kernel automatically closes files after the process terminates.

## 3.6 - `lseek` Function
- Every open file has an associated "current file offset", which is a non-negative integer measuring the number of bytes you are from the beginning of the file.
- Read and write operations use this offset to begin their operations, and change the offset accordingly based on number of bytes read/written.
- Offset is initialized to 0 when a file is opened (unless `O_APPEND` is specified, in which case it is the last character in the file).
- You can explicitly set this offset using `lseek`:
```c
#include <unistd.h>

off_t lseek(int fd, off_t offset, int whence);
/* Returns new file offset on success, -1 on error. */
```

- Offset is relative based on the value of `whence`. Let the offset given to the function call be `input_offset`:
    - If `whence` is `SEEK_SET`, then `new_offset = input_offset + 0` (beginning of file).
    - If `whence` is `SEEK_CUR`, then `new_offset += input_offset`.
    - If `whence` is `SEEK_END`, then `new_offset = file_size + input_offset` (this is usually used with a negative offset to be `x` bytes from the end of file).

- You can determine current offset by calling `lseek` with an offset argument of 0:
```c
off_t curr_pos = lseek(fd, 0, SEEK_CUR);
```

- Certain devices do allow negative offsets, but for regular files the offset must be non-negative. You should generally treat negative offsets as errors.
- The file's offset can be greater than the size of the file, in which case the next `write` operation will extend the file.
- **File hole**: Setting offset "out of file bounds" and then writing. Space in between previous end-of-file and current offset are filled with null characters (`\0`).
    - *IMPORTANT: File holes are not required to have storage backing it on disk. If you have a file with a "size" of 50 MB and you only have 45 MB of storage on disk, you can still theoretically store the file if there are large enough holes that do not require disk space.*

There's some stuff about system-specific settings and 32-bit/64-bit that I didn't take notes on.

## 3.7 - `read` Function
After opening a file, you can read data:
```c
#include <unistd.h>

ssize_t read(int fd, void *buf, size_t nbytes);
/* Returns number of bytes read on success, 0 if EOF, -1 on error. */
```
Reasons why you might read less bytes than requested:
- You hit EOF. (ex: `read(fd, buf, 100)` on a file with 30 bytes before EOF will return 30. Will return 0 on next call).
- You're reading from a terminal device, which gives you one line at a time (see chapter 18 for more).
- You're reading from a network with interruptions.
- You're reading from a pipe or FIFO with less bytes than what you requested.
- You're reading from a record-oriented device such as magnetic tape, which returns a single record at a time.
- You're interrupted by a signal.

The read operation begins at the file's current offset, which you can change with `lseek`.

## 3.8 - `write` Function
Once a file is open, you can write to it:
```c
#include <unistd.h>

ssize_t write(int fd, const void *buf, size_t nbytes);
/* Returns number of bytes written on success, -1 on error. */
```

Unlike `read`, you should usually be able to write the number of bytes requested; the only time it should fail is if you filled up disk space or you've exceeded the file size limit for a given process (see 7.11).

Write begins at the file's current offset unless `O_APPEND` was specified on `open`, in which case the offset is set to EOF before each write operation (and offset is incremented by number of bytes actually written).

## 3.9 - I/O Efficiency
Most file systems support some kind of read-ahead to improve performance on sequential reads; the system assumes you'll need the next information shortly and loads it into memory. Testing has shown that a buffer size above 4096 yields exponentially diminishing returns.

## 3.10 - File Sharing
*Note that most of the information here is conceptual and implementation varies across specific distributions.*
- UNIX can share files across different processes.
- Kernel uses 3 data structures to represent an open file. The relationships among them determine the effect one process has on another in regards to file sharing.

1. **Process Table**
    - Every process has an entry in the process table.
    - Each entry has a table of open file descriptors with one entry per descriptor.
    - Each file descriptor has the appropriate flags and a pointer to a file table entry.
2. **File Table**
    - Kernel has a file table for all open files. Each entry contains (a) The file status flags for the file, (b) the current file offset, and ( c) a pointer to the v-node table entry for the file.
3. **v-node Structure**
    - Each open file (or device) has a v-node structure containing information about the type of file and pointers to functions that operate on the file.
    - For most files, the v-node also contains the **i-node** for that file, which is read from disk on file open.
    - i-node contains file owner, size, pointers to actual data blocks on disk and more. See 4.14 for more details.

*Instead of splitting v-node data structures into seperate v-node and i-node, Linux uses a **system-independent** i-node and a **system-dependent** i-node.

- Each process has its own file table entry with its own current file offset. However, unexpected results can arise when multiple processes write to the same file.

## 3.11 - Atomic Operations
### Appending to a File

### `pread` and `pwrite` Functions

### Creating a File

## 3.12 - `dup` and `dup2` Functions

## 3.13 - `sync`, `fsync`, and `fdatasync` Functions

## 3.14 - `fcntl` Function

## 3.15 - `ioctl` Function

## 3.16 - `/dev/fd`

## 3.17 - Summary
