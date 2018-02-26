# Chapter 4: Files and Directories

## 4.1 - Introduction
No notes taken.

## 4.2 - `stat`, `fstat`, `fstatat`, and `lstat` Functions
This chapter centers on these 4 functions and the info they return:
```c
#include <sys/stat.h>

int stat(const char *restrict pathname, struct stat *restrict buf);
int fstat(int fd, struct stat *buf);
int lstat(const char *restrict pathnamee, struct stat *restrict buf);
int fstatat(int fd, const char *restrict pathname, struct stat *restrict buf, int flag);

/* All return 0 on success, -1 on error. */
```

- `stat` returns a struct of info about the named file *path*.
- `fstat` gets similar info about an open file descriptor.
- `lstat` is like `stat`, but when the file is a symbolic link, it returns information about the symbolic link file and not the file it links to (*so use this one over `stat` usually*).
- `fstatat` is like `fstat` but it's a relative pathname and you can give it flags, such as whether or not to follow symbolic links.
- You must supply the `buf` argument simply by passing in a struct of the value `struct stat`:
```c
if (lstat(path, &buf) < 0) {
    /* Error handling */
}
```
- See the textbook for what the contents of the struct are.
- The `timespec` struct has time in seconds and nanoseconds.
- The biggest user of the `stat` functions is the `ls` command.

## 4.3 - File Types
Most files in a UNIX system are either regular files or directories, but there are more. Here is the full list:
1. **Regular File**: Contains data of some form. Kernel does not care whether it is text or binary; any interpretation is left to applications processing the file (*exception is binary executables, which must conform to a certain standard so kernel knows how to execute them*).
2. **Directory File**: Contains names of other files and pointers to information on these files.
    - Any process with read permissions on a directory can read contents, *but only kernel has write permissions*.
    - Processes must go through system functions to make changes to a directory; cannot change directly!
3. **Block Special File**: Provides buffered I/O access in fixed-size units to devices such as disk drives.
4. **Character Special File**: Provides unbuffered I/O access in variable-size units to devices. All devices on a system are either BSFs or CSFs.
5. **FIFO**: Used for communication between processes, sometimes called a **pipe**. See 15.5 for more information.
6. **Socket**: Used for network communication between processes (or non-network communication on a single host). See Chapter 16 for more information.
7. **Symbolic Link**: Points to another file. See 4.17.

- The stat structure's `st_mode` field encodes the filetype, and we can use macros in `<sys/stat.h>` such as `S_ISREG()` or `S_ISLNK()` to get that info.
- The thing about semaphores is omitted cuz I have no idea what they're talking about yet, so see Chapter 15.
- *Fun fact: You can break a terminal line on `\` just like in C.*

## 4.4 - Set-User-ID and Set-Group-ID
Every process has 6 or more IDs associated with it:
- Who we really are: real user ID, real group ID
    - Taken from password file, don't usually change during a login session, though there are ways to change this with a superuser process.
- File access permission checks: effective user ID, effective group ID, supplementary group ID
    - This usually equals real user ID and real group ID.
- Saved by `exec` functions: Saved set-user-ID, saved set-group-ID.
    - Are copies of effective user ID and effective group ID created when a program is executed.

Every file has an owner and group owner. Owner is in `st_uid` member of stat, and group owner is in `st_gid`.
- You can set a flag in the file's `st_mode` that says "on execution, change effective user ID to be the owner of the file (`st_uid`) or the group owner of the file (`st_gid`). These flags are called the set-user-ID bit and the set-group-ID bit.
- If the owner of a file is the superuser and the file's set-user-ID bit is set, then you can run that process with superuser priviliges regardless of real user ID. An example is the program to change your password, which requires superuser priviliges even though the current logged-in user may not have those priviliges.

## 4.5 - File Access Permissions
- A file's permission bits are also stored in `st_mode`. Any and every file has permissions, including symbolic links and directories.
- There are 3 categories of permissions: user, group, and other. Each has read, write, and execute bits, which can be obtained by masking the permission bits with masks like `S_IWGRP` (group-write):
```c
int can_i_write_from_group = bits & S_IWGRP;
```

Rules for permissions
- To open any type of file by name, you must have execute permission in each directory in the absolute filepath and the appropriate permission for the file based on what you're trying to do with it (read, write, and/or execute).
    - To summarize, you need read permission to see the contents of a directory, and you need execute permission to "pass through" it (which is why the execute bit of a directory is sometimes called the "search bit")
- Read permission for a file lets us open an existing file for reading with the flags `O_RDONLY` or `O_RDWR`
- Write permission is similar to above with flags `O_WRONLY`, `O_TRUNC`, and `O_RDWR`.
- To create or delete a file in a directory, you must have write and execute permission in the directory. You do not need permissions for the file itself.
- To execute a file, filetype must be regular and you must have execute permissions.

Each time a process opens, creates, or deletes a file, the kernel performs file access tests that depend on the owners of the file, the effective IDs of the process, and the supplementary group IDs of the process (if supported).
- Owner IDs are properties of the file
- Effective IDs and Supplementary Group IDs are properties of the process.

The kernel tests go in this order and end once a test passes:
1. If effective user ID is 0 (superuser), no tests are required; superuser has free rein over system.
2. If effective user ID is the same as owner ID of the file (so the process therefore owns the file), access is granted based on the appropriate user permission bit (rwx).
3. If effective group ID or one of the supplementary group IDs of the process equals the group ID of the file, then access is granted based on the appropriate group permission bit (rwx).
4. Otherwise, check the "other" rwx permission bits and grant access based on those.

## 4.6 - Ownership of New Files and Directories
The following rules describe values assigned to user ID and group ID of a new file created using either `open` or `creat` (or a directory using `mkdir`):
1. User ID is set to effective user ID of the process
2. Group ID is set to one of the following:
    a. Effective group ID of the process.
    b. Group ID of its parent directory. This ensures that all files and directories created in the specified parent directory have the same group ID.

## 4.7 - `access` and `faccessat` Functions
Sometimes when opening a file, a process wants to test accessibility based on the real user and group IDs instead of going through the kernel tests. This is useful when a process is running as someone else using either the set-user-ID or the set-group-ID feature.

*(For example, even if a process uses set-user-ID to root, it might still want to verify that the real non-superuser can still access a file)*

The `access` and `faccessat` functions test permissions based on the real user and group IDs. They follow the same kernel steps outlined in 4.5; just replace the word "effective" with the word "real":
```c
#include <unistd.h>

int access(const char *pathname, int mode);
int faccessat(int fd, const char *pathname, int mode, int flag);

/* Both return 0 on success, -1 on error. */
```

The `mode` field is either `F_OK` to test if a file exists, or a bitwise OR of any of the following flags:
    - `R_OK`: read permission granted
    - `W_OK`: write permission granted
    - `X_OK`: execute permission granted

The `faccessat` function and the `access` functions are identical in function except that `faccessat` takes a relative path.

You can set the `AT_EACCESS` flag in `faccessat` to use effective user/group IDs instead of the real user/group IDs.

## 4.8 - `umask` Function
Sets the file mode creation mask for the process and returns the previous value. There is no error value.
```c
#include <sys/stat.h>
mode_t umask(mode_t cmask);
```

The `cmask` argument is the bitwise OR of any of the nine constants `S_IRUSR`, `S_IWUSR`, etc.

The **creation mask** is used whenever the process creates a new file/directory (and `open`/`creat` both accept a `mode` argument).

Most users never deal with their `umask` value; it is usually set once on login by the shell's startup file. Note that changing the `umask` value for a process does not change the mask of its parent (usually the shell). You can change the value as a "temporary default" when creating new files with a process. This value is expressed in octal. The first bit is user, the second is group, and the third is other. A bit-value of 4 means read, 2 means write, and 1 means execute. 

*Note that this is not usual adding/subtracting--this is turning bits on and off in the octal values!*

For example, a umask of 027 means that the user has all permissions, the group has write and execute, and other has no permissions.

## 4.9 - `chmod`, `fchmod`, `fchmodat` Functions
These functions allow a process to change access permissions for an existing file:
```c
#include <sys/stat.h>

int chmod(const char *pathname, mode_t mode);
int fchmod(int fd, mode_t mode);
int fchmodat(int fd, const char *pathname, mode_t mode, int flag);
/* All return 0 on success, -1 on error. */
```
Note that `chmod` does not require the file to be open, whereas the other two require file descriptors from an open file.

To change permission bits of a file, the effective user ID of the process must be equal to the owner ID of the file, or the process must have superuser permissions.

The `mode` field is a bitwise OR of the constants for `chmod` functions, such as `S_IRGP` (read by group).

## 4.10 - Sticky Bit
The `S_ISVTX` bit was used differently in the past than it is now. In the past, if it was set for an executable program file, then the first time the program was executed, a copy of the program's text (machine instructions) was saved in the swap area on termination, which made it load faster next time since it was handled as a contiguous file. This was often done for common applications like the text editor or passes of the C compiler. Newer UNIX systems have a virtual memory system and a faster file system, so this is no longer necessary.

Now, the sticky bit can be set for a file or directory. If the sticky bit is set for a directory, any file in the directory can be removed or renamed only if the user has write permissions and either:
a. Owns the file,
b. Owns the directory, or
c. Is the superuser

This is often seen in the `/tmp` and `var/tmp` directories.

## 4.11 - `chown`, `fchown`, `fchownat`, and `lchown` Functions
We saw that `chmod` (**ch**ange **mod**e) changes the mode (permission bits). Here we can use variations of `chown` (**ch**ange **own**er) to change a file's user ID and group ID.
```c
#include <unistd.h>

int chown(const char *pathname, uid_t owner, gid_t group);
int fchown(int fd, uid_t owner, git_t group);
int fchownat(int fd, const char *pathname, uid_t owner, gid_t group, int flag);
int lchown(const char *pathname, uid_t owner, gid_t group);
/* All return 0 on success, -1 on error. */
```
*Note that if either of the arguments `owner` or `group` is -1, the corresponding ID is left unchanged.*

- These functions operate similarly unless the referenced file is a symbolic link, in which case `lchown` and `fchownat` change the owners of the link itself instead of the file pointed to by a symbolic link.
- Since `fchown` operates on a file that is already open, it can't be used to change the ownership of a symbolic link.
- When `_POSIX_CHOWN_RESTRICTED` is in effect, you can't change the user ID of your files.
- If these functions are called by a process other than a superuser process, on seccessful return, both the set-user-ID and the set-group-ID bits are cleared.

## 4.12 - File Size
- The `st_size` member of the `stat` structure contains the size of the file in bytes. It only means something for regular files, directories, and symbolic links. *Some OS's also defien file sizes for pipes.*
- A file size of 0 is allowed only for a regular file.
- Directory file sizes are usually multiples of a number such as 16 or 512.
- A symbolic link's file size is the number of bytes in the path. So, if a symbolic link points to `usr/lib`, then the file size is strlen("usr/lib") = 7.
- Most contemporary UNIX systems provide `st_blksize` as the preferred block size for read/write operations, and `st_blocks` as the number of 512-byte blocks that are allocated. *Note that some systems may not use 512.*

### Holes in a File
Holes are created by seeking past the current EOF and writing some data. You can use the `du` command to see your current storage space, and you can store files "larger" than the amount of storage space you have if the actual required hard disk space (size - size_of_holes) is within bounds. Holes are filled with 0 bytes.

## 4.13 - File Truncation
Sometimes you want to chop off the rest of a file after a specified point. Emptying a file is defined as truncating from the beginning and is done with the `O_TRUNC` flag when opening a file (recall that the offset when opening a file is 0, so you will effectively truncate from the beginning).

The following two functions truncate an existing file to `length` bytes:
```c
#include <unistd.h>

int truncate(const char *pathname, off_t length);
int ftruncate(int fd, off_t length);
/* Both return 0 on success, -1 on error. */
```

- If the previous size of the file was greater than `length` (truncating "within bounds"), then any data after offset `length` is no longer accessible.
- If the previous filesize was less than `length`, then the file size will increase and any data between the old EOF and the new EOF is filled with `\0` bytes, creating a hole.

