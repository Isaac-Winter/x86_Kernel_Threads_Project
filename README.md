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
