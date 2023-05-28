---
title: IDA Challenge - Free Madame de Maintenon
date: 2023-05-28
categories: [Reversing, IDA]
tags: [ida]    
---

# Background

Recently [Hex Rays](https://hex-rays.com/) released a reversing challenge called "Free Madame de Maintenon" and this is a brief write up that documented the process of solving it.

![IDA Twitter](/assets/img/ida-twitter.png)

# Challenge

## File

```bash
➜ file challenge
challenge: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=83db7b297901c743a71f43e813e3dc266245b220, for GNU/Linux 3.2.0, stripped

```

## Protections

```bash

➜ checksec 
[*] '/home/careless/Desktop/play-ground/ctf/Others/challenge'
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
    FORTIFY:  Enabled

```

## Symbol Table

```bash
➜ objdump -T challenge         

challenge:     file format elf64-x86-64

DYNAMIC SYMBOL TABLE:
0000000000000000      DF *UND*	0000000000000000  Base        SDL_DestroyWindow
0000000000000000      DF *UND*	0000000000000000  Base        SDL_Quit
0000000000000000      DF *UND*	0000000000000000  Base        SDL_RenderPresent
0000000000000000      DF *UND*	0000000000000000  Base        SDL_CreateWindowAndRenderer
0000000000000000      DF *UND*	0000000000000000  Base        SDL_DestroyRenderer
0000000000000000      DF *UND*	0000000000000000 (GLIBC_2.2.5) strncpy
0000000000000000      DF *UND*	0000000000000000  Base        SDL_PollEvent
0000000000000000      DF *UND*	0000000000000000 (GLIBC_2.34) __libc_start_main
0000000000000000      DF *UND*	0000000000000000  Base        SDL_Init
0000000000000000      DF *UND*	0000000000000000  Base        IMG_Init
0000000000000000      DF *UND*	0000000000000000  Base        SDL_RenderClear
0000000000000000      DF *UND*	0000000000000000 (GLIBC_2.4)  __stack_chk_fail
0000000000000000      DF *UND*	0000000000000000  Base        SDL_RenderCopy
0000000000000000      DF *UND*	0000000000000000  Base        SDL_RWFromConstMem
0000000000000000      DF *UND*	0000000000000000  Base        IMG_LoadTexture_RW
0000000000000000      DF *UND*	0000000000000000 (GLIBC_2.3.4) __fprintf_chk
0000000000000000  w   D  *UND*	0000000000000000  Base        _ITM_deregisterTMCloneTable
0000000000000000  w   D  *UND*	0000000000000000  Base        __gmon_start__
0000000000000000  w   D  *UND*	0000000000000000  Base        _ITM_registerTMCloneTable
0000000000000000  w   DF *UND*	0000000000000000 (GLIBC_2.2.5) __cxa_finalize
000000000015df60 g    DO .bss	0000000000000008 (GLIBC_2.2.5) stderr
0000000000001220 g    DF .text	00000000000002dc  Base        main
```

## Run

```bash
➜ ./challenge 
You need to give the password as the argument%
```

With Password:

```bash
➜ ./challenge 1234
```

![Failed](/assets/img/ida-challenge-failed.png)

## IDA & Disassembly

![IDA Disassembly](/assets/img/ida-challenge-disassembly.png)

This program expects the password with the length of **24 (mov edx, 18h)** and performs some operations and checks on it. The result of such operations are used with what I would call it "image" buffer in .data section in order to produce the final image. The goal here is to find the password that produces the "correct" image.

## Operations

These are (roughly) the necessary conditions to meet in order to reach the "success" block:


```c

// Condition 1
if((flag[16:18] + flag[22:24] - flag[8:10] - flag[14:16]) != 0x1cd4){
	JMP_TO_FAIL;
}
// Condition 2
if ((flag[20:22] + flag[6:8] + flag[2:4] - flag[10:12]) != 0xd899){
	JMP_TO_FAIL;
}
// Condition 3
if ((flag[16:] ^ flag[0:8]) != 0xA04233A475D1B72){
	JMP_TO_FAIL;

}
// Condition 4
while (counter != 0x1d9AD){
	buffer[8 * counter] -= flag[counter] - ((0xAAAAAAAAAAAAAAAB * counter >> 64) >> 1) + (0xAAAAAAAAAAAAAAAB * counter >> 64) & 0xFFFFFFFFFFFFFFFE)
	++counter;
}

if (flag[20:24] + (flag[0:4] * 2) - (4 * flag[8:12]) - (flag[16:20] >> 3) - (flag[4:8] >> 3) != 0x4B5469C){
	JMP_TO_FAIL;	
}

 
// Condition 5
idx = 0
counter = 1;
while (counter <= 0x3B35A){
	&buffer[4 * idx] ^= flag;
	counter += idx;
	offset = (offest + 1) % 6
	idx = counter;
}

if((flag[8:16] ^ flag[16:]) != 0x231F0B21595D0455){
	JMP_TO_FAIL;
}

```

## Initial Attempt with Z3

```python

from z3 import *

flag = [BitVec(f"flag_{i:02}",16) for i in range(0,12)]
s = Solver()

s.add((flag[8] + flag[-1]-flag[4]-flag[7]) == 0x1cd4)
s.add((flag[3] + flag[1] + flag[-2] - flag[5]) == 0xd899)

if repr(s.check()) == "sat":
    m = s.model()

solution = sorted([(d, m[d].as_long()) for d in m], key=lambda x: str(x[0]))
print (b''.join([struct.pack("<H",int(str(x[1]),10)) for x in solution]))
...
```

Result:

```bash
➜ python solver.py
b'/\x08J\x11\x01\x08\x88\x02\x9d\x03\x99p\x01@aA\x84\x01l\x08`5N`'
```
It was not clear to me at the time of writing (now I do know for sure) whether it was possible to modify the size of what was originally defined in BitVec (for 64bit xor in the challenge). This was a good refresher on some of the quirks and available functions in z3. Regardless, I spent a long enough time on z3 and moved onto other method.


## Angr

Angr, which is every CTF player's go-to tool for symoblic execution, is a great choice for this case. It is still necessary to provide the constraints and such but at least it would steps through every basic blocks and find the input that satisfies the condition (if it exists) for me. However, there are a few things to note for this challenge:

```bash
1. Argv[1],which is our password,does get copied into some buffer on the stack. It is recommended to load our BitVec via hooks directly into the password buffer ([rsp+0x50]). Also, make sure to remove \n,\r,\t,[space] characters for strncpy.
2. There is a check whether SDL_CreateWindowAndRenderer was successful or not (by checking the returned value greater than 1). Since I have excluded all SDL related functions via exclude_sim_procedures_list, I hooked into the address right after the call to set its value to greater than 1.
3. Patch the block that loads the failed image (offset 0x12F7) via SDL_RWFromConstMem. This is done to make sure it does not poll and simply exits when it fails.
4. There is an issue with loading the address of image buffer to rbp (offset 0x13d0) for some reason. I got around by manually setting the register via hooks.
```

With the above notes in mind, this is the solution script I came up with for the challenge:

```python
import angr
import claripy


project = angr.Project("./challenge", auto_load_libs=False, exclude_sim_procedures_list=["strncpy","SDL_Init","IMG_Init","SDL_CreateWindowAndRenderer"])
argv1 = claripy.BVS("argv1",24*8)

#0x400000
def check(state):
    if state.ip.args[0] == 0x401272:
        state.regs.rdx = 2

    if state.ip.args[0] == 0x4012a5:
        state.memory.store(state.regs.rsp+0x50,argv1)


@project.hook(0x401272)
def set_ebx(state):
    state.regs.rdx = 0x2

@project.hook(0x4012a5)
def set_ebx(state):
    state.memory.store(state.regs.rsp+0x50,argv1)

@project.hook(0x4013d0)
def set_buffer(state):
    state.regs.rbp = 0x400000 + 0x711e0

@project.hook(0x4014e2)
def write_buffer(state):
    data = state.memory.load(0x400000 + 0x711e0,506764)
    buffer = state.solver.eval(data,cast_to=bytes)
    with open("x.png","wb") as f:
        f.write(buffer)

init_state = project.factory.entry_state(args=["./challenge","1"*24])

for x in argv1.chop(8):
    init_state.solver.add(x != 0x20)
    init_state.solver.add(x != 0x0a)
    init_state.solver.add(x != 0x0d)
    init_state.solver.add(x != 0x9)
    init_state.solver.add(x > 0)
    init_state.solver.add(x != 0)
    init_state.solver.add(x > 0x20)
    init_state.solver.add(x < 0x7f)


sm = project.factory.simulation_manager(init_state)

if len(sm.found) > 0:
    print (len(sm.found))
    print (sm.found)
    print (sm.deadended)
    for x in sm.found:
        a = x.solver.eval(argv1, cast_to=bytes)
        payload = ""
        for i in a:
            payload += f'\\x{i:02x}'
        print (payload)
else:
    print ("No Solution Found!")

```

It does take a bit to run but eventually it shows the following password:

```bash
➜ python solver.py
\x46\x72\x33\x33\x5f\x4d\x34\x64\x61\x6d\x33\x2d\x44\x65\x2f\x4d\x34\x69\x6e\x74\x65\x6e\x30\x6e #Fr33_M4dam3-De/M4inten0n
```

Verifying the above password with the challenge:


![Success](/assets/img/ida-challenge-success.png)

# Conclusion

It was a little fun challenge to poke at and thanks to Hex Rays team for putting it out. I am hoping to see more fun challenges in the future :) 


