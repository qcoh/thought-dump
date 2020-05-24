JIT Compiler for a GameBoy Emulator
-----------------------------------

These are just some unfiltered thoughts. Probably lots of nonsense.

## Indirect reading and writing

Initially, to make things simple for myself, all reading from and writing to the mmu
is done indirectly via the registers r10 and r11, where r10 is always used as the
address and r11 the source or destination.

Example: Instead of emitting

```
mov al, BYTE [0x1234]
```

emit

```
mov r10, 0x1234
mov r11b, BYTE [r10]
mov al, r11b
```

The upside of this approach is that in the signal handler for SIGSEGV, decoding the
instruction becomes much more simple: We only need to compare three or four bytes
at the instruction pointer with the four, legal reads and writes:

* mov r11b, BYTE [r10]: 0x45 0x8a 0x1a
* mov r11w, WORD [r10]: 0x66 0x45 0x8b 0x1a
* mov BYTE [r10], r11b: 0x45 0x88 0x1a
* mov WORD [r11], r11w: 0x66 0x45 0x89 0x1a

## Virtual memory, twice

It would be nice, and perhaps performance-relevant, if we didn't have to mprotect,
read/write, mprotect whenever a SIGSEGV is raised.

This can be achieved by first memory mapping a file with fine-grained PROT_READ,
PROT_WRITE permissions: This is what the emulated CPU gets.

A second memory mapping with all PROT_READ | PROT_WRITE is created and used by
the signal handler. When we encounter a SIGSEGV for writing to, say, the hardware
registers, we can handle the event in the signal handler and set the value by
writing the value to the second memory mapping.

Reference: https://stackoverflow.com/a/25401802
