# Chapter 8: Process Control

## 8.1 - Introduction
No notes taken

## 8.2 - Process Identifiers
- Every process has a unique process ID (non-negative int). Is always unique, and therefore used as a piece of other identifiers to guarantee uniqueness.
  - Ex: Applications sometimes put process ID in filename to get unique filenames.
- Process IDs are reused when a process terminates. UNIX usually delays reuse of a recently-used process to prevent identification mistakes.
- Process ID 0 is usually the **scheduler** process, or **swapper**. This is part of the kernal and is known as a system process.
- Process ID 1 is usually the **`init`** process and is invoked by the kernel at the end of the bootstrap procedure.
  - **Bootstrapping** is the process of starting up a computer from a halted or powered-down condition.
  - Reads system-dependant initialization files and brings the system to a certain state.
  - Never dies and is a normal user process with superuser priviliges.
  - Becomes the parent process of any orphaned child process.
  - MacOS has replaced `init` with `launchd`, which does the same thing with expanded functionality.
- Each UNIX system implementation has its own set of kernel processes that provide OS services.
  - Ex: On some virtual memory implementations, Process ID 2 is the **pagedaemon**, which is responsible for supporting the paging of virtual memory systems.

The following getter functions in `<unistd.h>` retrieve certain process identifiers. None of them return errors:

```c
#include <unistd.h>

pid_t getpid(void); /* Returns process ID of calling process. */

pid_t getppid(void); /* Returns process ID of parent of calling process. */

uid_t getuid(void); /* Returns real user ID of calling process. */

uid_t geteuid(void); /* Returns effective user ID of calling process. */

gid_t getgid(void); /* Returns real group ID of calling process. */

gid_t getegid(void); /* Returns effective user ID of calling process. */
```

## 8.3 - `fork` Function
- An existing process can create a new process with the `fork` function:

```c
#include <unistd.h>

pid_t fork(void);

/* Returns 0 in child process, pid of child in parent process, -1 on error. */
```

- New process is called **child process**, a copy of the parent process.
  - Child gets a copy of the parent's data space, heap, and stack. *Note that these are copies, and the two processes do not actually share the same memory space; rather, the same amount of memory used by the parent is reserved for the child.*
  - The child does share the text segment (code).
- Function returns twice, once in the parent and once in the child.
  - Returns to the parent because a process can have multiple children, and there's no function that gets a list of all children.
  - Returns 0 to child because a process can only have one parent, and a child can always call `getppid` to get the process ID of the parent. Since Process ID 0 is reserved for the kernel, it's not possible for 0 to be the Process ID fo a child.

### Example
See how changes to variables in a child process do not affect values in the parent process:

```c
#include "apue.h"

int glob = 6;       /* external variable in initialized data */
char buf[] = "a write to stdout\n";

int main(void) {
    int var;      /* automatic variable on the stack */
    pid_t pid;

    var = 88;
    
    if (write(STDOUT_FILENO, buf, sizeof(buf) - 1) != sizeof(buf) - 1) {
        err_sys("write error");
    }
    
    printf("before fork\n");    /* we don't flush stdout */

    if ((pid = fork()) < 0) {
        err_sys("fork error");
    } 
    else if (pid == 0) {      /* Within child process */
        glob++;               /* modify variables */
        var++;
    } 
    else {
        sleep(2);             /* Within parent process */
    }

    printf("pid = %d, glob = %d, var = %d\n", getpid(), glob, var);
    exit(0);
}
```

```
OUTPUT
$ ./a.out
a write to stdout
before fork
pid = 430, glob = 7, var = 89   *Child's variables were changed*
pid = 429, glob = 6, var = 88   *Parent's copy was not changed*
$ ./a.out > temp.out
$ cat temp.out
a write to stdout
before fork
pid = 432, glob = 7, var = 89 
before fork
pid = 431, glob = 6, var = 88
```

- We never know whether the child starts executing before the parent, or vice versa; this depends on the kernel's scheduling system. If synchronous operations are necessary, there must be some form of interprocess communication. 
  - In the code above, parent is put to sleep for 2 seconds so that child executes; however, there is no guarantee that this is a long enough waiting period.
  - See Section 10.16 to see how signals can be used to synchronize a parent and child after a `fork`.
- Note that `write` is not buffered, and since it is called before `fork`, its data is written ocne to stdout.
- However, the standard I/O library *is* buffered. When we run the program, we only get a single copy of the first `printf` line because stdout is flushed by `\n`.
- When we redirect stdout to a file, we get two copies of the `printf` line, and `fork` does not flush the buffer. Therefore, both the parent and child have a buffer, and the second `printf` before the exit just appends its data to the existing bufer. The buffer copies in the parent and child are only flushed when the process terminates.
  - This is why you should explicitly `fflush` stdout.

### File Sharing

## 8.4 - `vfork` Function

## 8.5 - `exit` Functions

## 8.6 - `wait` and `waitpid` Functions

## 8.7 - `waitid` Function

## 8.8 - `wait3` and `wait4` Functions

## 8.9 - Race Conditions

## 8.10 - `exec` Functions

**I did not continue after this section as it was not immediately relevant to the course.**
