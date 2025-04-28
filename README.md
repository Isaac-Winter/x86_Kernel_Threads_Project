# x86_Kernel_Threads_Project
### Create System Calls and a Thread Library to Deploy Kernel Threads in QEMU x86, through WSL.
## Running xv6 through QEMU in WSL (WSL2)

`$sudo apt update`

`$sudo apt upgrade`

`$sudo apt install build-essential qemu-system-x86 gdb git`

`$git clone https://github.com/mit-pdos/xv6-public.git`

If git clone stalls or fails, restart WSL by opening host CMD as administrator, run powershell, then: `wsl --shutdown`

`$sudo apt install qemu-system-x86`

Verify install with `qemu-system-i386 --version`

`$cd xv6-public`

`$make`

If you get errors around this point, try: `$nano Makefile`
ctrl w to search for `CFLAGS` then remove `-Werror` from that line. Save and Exit. Clean with: `make clean` then do:
`$make qemu`.
This should open a QEMU window. You can exit it again as you need to write the xv6 code outside of QEMU, which is basically just a simple operating system to test kernel or OS related code.

### To make a test program:
In /xv6-public: `$nano hi.c`

Copy this code into the file. Note: the imported libraries may vary, check the contents of your /xv6-public directory with "ls" if errors related to the imports are thrown.

```c
#include “types.h”
#include “user.h”
int main (void) {
	printf(1, “hi from xv6\n”);
	exit();
}
```
Save and exit. Then:
`nano Makefile`

Find the list of user programs in this file, should be around line 220, or ctrl w for `UPROGS`
Add the hi.c program to this list by adding  `_hi\` to the list. 

NOTE: syntax in this file is very delicate and will cause errors if spacing is fiddled with. Make sure to tab over `_hi\` instead of using spaces, and make sure you don’t add any spaces, indents, or blank lines anywhere. If so, you’ll get an error message:

`Makefile:(the line # of the error): *** recipe commences before first target. Stop.`

Or something along those lines.

Save and exit Makefile.

Then:

`$make clean`

`$make qemu`

In QEMU terminal, run the hi.c code with:
`hi`

You should see:
`hi from xv6`

## Adding Kernel Threads
### Creating and testing clone() and join() system calls.
The word doc Updated Steps for Kernel Threads Project contains file by file instructions on how to create the new system calls and test them with the `clone_test.c` program.

Since you may prefer to copy the code from this readme instead of a word doc, I will include any code that is more than a single line.

### **In sysproc.c** 
Add the following to the top of the file, after the imports.

```c
extern int join(void **);

int
sys_join(void)
{
  void *stackp;
  if (argptr(0, (char**)&stackp, sizeof(void *)) < 0)
    return -1;
  return join((void **)stackp);
}

int
sys_clone(void)
{
  uint fcn, arg1, arg2, stack;

  if(argint(0, (int*)&fcn) < 0 || argint(1, (int*)&arg1) < 0 || argint(2, (int*)&arg2) < 0 || argint(3, (int*)&stack) < 0)
    return -1;

  return clone((void (*)(void*, void*))fcn, (void*)arg1, (void*)arg2, (void*)stack);
}
```
### **In proc.c**
Put clone() code into the bottom of proc.c

```c
int clone(void (*fcn)(void*, void*), void *arg1, void *arg2, void *stack) {
  int i;
  struct proc *np;
  struct proc *curproc = myproc();

  // Stack must be page-aligned and non-null
  if ((uint)stack % PGSIZE != 0 || stack == 0)
    return -1;

  // Allocate process (thread)
  if ((np = allocproc()) == 0)
    return -1;

  // Share address space
  np->pgdir = curproc->pgdir;
  np->sz = curproc->sz;
  np->parent = curproc;

  // Copy trapframe from parent
  *np->tf = *curproc->tf;

  //So child sees 0 return from syscall
  np->tf->eax = 0;

  // Set up new user stack
  uint sp = (uint)stack;

  // Push arg2
  sp -= 4;
  *(uint*)sp = (uint)arg2;

  // Push arg1
  sp -= 4;
*(uint*)sp = (uint)arg1;

  // Push return address
  sp -= 4;
  *(uint*)sp = (uint)exit;

  // Set the stack pointer for the child process to the top of the stack (where args are located)
  np->tf->esp = sp;

  // Set instruction pointer to thread function, which is where the child will start executing
  np->tf->eip = (uint)fcn;

  // Save the user stack for the join function
  np->stack = stack;

  // Share open files with the child
  for(i = 0; i < NOFILE; i++) {
    if(curproc->ofile[i])
      np->ofile[i] = filedup(curproc->ofile[i]);
  }
  np->cwd = idup(curproc->cwd);

  safestrcpy(np->name, curproc->name, sizeof(curproc->name));

  // Mark the new process as runnable
  np->state = RUNNABLE;

  return np->pid;
}
```

At the bottom of proc.c, under `clone()`, add `join()`
```c
int
join(void **stackp)
{
  struct proc *p;
  int havekids;
  struct proc *curproc = myproc();

  acquire(&ptable.lock);

  for(;;) {
    havekids = 0;
    for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
      if(p->parent != curproc || p->pgdir != curproc->pgdir)
        continue;
      havekids = 1;
      if(p->state == ZOMBIE){
        int pid = p->pid;

        if (stackp)
          *stackp = p->stack;   // Retrieve the saved user stack pointer

        // Free resources (like wait())
        kfree(p->kstack);
        p->kstack = 0;
        p->state = UNUSED;
        p->pid = 0;
        p->parent = 0;
        p->name[0] = 0;
        p->killed = 0;
        p->stack = 0;          // Clear saved stack pointer

        release(&ptable.lock);
        return pid;
  }
    }

    if(!havekids || curproc->killed){
      release(&ptable.lock);
      return -1;
    }

    sleep(curproc, &ptable.lock);
  }
}
```
## **clone_test.c**
A very simple program to test that threads are being created. To be able to run it in QEMU, follow the same steps as with the hi.c program.
```c
#include "types.h"
#include "stat.h"
#include "user.h"

#define STACK_SIZE 4096

void thread_func(void *arg1, void *arg2) {
  int a = *(int*)arg1;
  int b = *(int*)arg2;
  printf(1, "Thread running: arg1 = %d, arg2 = %d\n", a, b);  // Changed printf to use file descriptor
  exit(); // No arguments in x86 xv6
}

int
main(int argc, char *argv[])
{
  void *stack;
  int arg1 = 42;
  int arg2 = 24;

  // Allocate a page for the user stack
  stack = malloc(STACK_SIZE);
  if (stack == 0) {
    printf(1, "Failed to allocate stack\n");  // Changed printf to use file descriptor
    exit();
  }

  // Ensure page-aligned stack
  if ((uint)stack % STACK_SIZE != 0) {
    stack = (void *)((uint)stack + (STACK_SIZE - ((uint)stack % STACK_SIZE)));
  }

  // Call clone with correct arguments
  int pid = clone(thread_func, &arg1, &arg2, (void*)((char*)stack + STACK_SIZE));
  if (pid < 0) {
    printf(1, "clone failed\n");  // Changed printf to use file descriptor
    free(stack);  // cleanup if clone fails
    exit();
  }

  printf(1, "Parent: created thread with pid %d\n", pid);  // Changed printf to use file descriptor

  wait();
  free(stack);
  printf(1, "Parent: thread finished\n");  // Changed printf to use file descriptor

  exit();
}
```
## Thread Library Creation and Final Test
This part of the project was completed by my group partner, Cyrus Amelunke, and **I am not taking credit for the contents of the CS380 Finalized thread test WSL QEMU assignment document**. If you want to implement that code as well, you can download the word doc and follow the steps.
