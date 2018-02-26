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



