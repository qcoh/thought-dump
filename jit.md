JIT Compiler for a GameBoy Emulator
-----------------------------------

These are just some unfiltered thoughts. Probably lots of nonsense.

# Using C++ Types

I want to have a DSL in C++ for writing assembly. Something along the lines of

```
const auto code = Assember()
| mov(rax, rbx)
| xor(rbx, rbx)
;
```

Moreover, I want to do as much as possible during compilation and use "advanced"
C++ features in a sensible, obvious way.

With variadic templates:

```
using u8 = unsigned char;

template <u8... Bytes>
struct assembler {
    static constexpr int size = sizeof...(Bytes);

    template <u8... ArgBytes>
    constexpr auto operator|(const assembler<ArgBytes...>&) const -> assembler<Bytes..., ArgBytes...> {
        return {};
    }

    constexpr auto emit() -> std::array<u8, size> {
        return {{ Bytes... }};
    }
};

struct rax_type {} rax;

constexpr auto inc(rax_type) -> assembler<0x48, 0xff, 0xc0> { return {}; }
```

These can be composed quite well. 

# Calling Conventions and GameBoy registers

The GameBoy's CPU is a derivative of the Zilog Z80, which is a clone (?) of the Intel 8080. This means that it should
be possible to map a lot of instructions to the host CPU. To do so, we first map the registers

GameBoy | Host (x86\_64)
|---|-------|
f | ah
a | al
af | ax
c | bh
db | bl
bc | bx
e | ch
d | cl
de | cx
l | dh
h | dl
hl |dx

In C++ the GameBoy's registers are stored in a struct

```
struct CPU {
    u8 af;
    u8 bc;
    u8 de;
    u8 hl;
};
```

The calling convention in Linux lets us pass arguments to functions via rdi, rsi, rdx, rcx, r8 and r9. Unfortunately, for all but rdx and rcx, there is register coinciding with the second lowest byte. So, when we call a jitted function

```
CPU cpu;
jitted_function(cpu.af, cpu.bc, cpu.de, cpu.hl);
```

We have to do something like this (TODO: check this):

```
push rbx ; rbx must be saved by the callee
mov rax, rdi
mov rbx, rsi
; jitted code
pop rbx
```

TODO: check if using the 32bit registers produces smaller code

# Calling JITted code

I once read that few games on the GameBoy use interrupts for handling input and instead read the hardware register value periodically. However, other interrupts are very timing sensitive.

To handle the blanking interrupts that occur every n cycles, I will simply not call the jitted code if its lenghts (in cycles) is longer than the time to the next interrupt.

To handle user input, jitted functions will have a maximum length of 64 (128?) bytes and will not include control-flow instructions (jp, call, ret).

# Fastmem

The page size on x86\_64 Linux is 4096 bytes. We can mark certain parts of the GameBoy's memory read-only (ROM banks!!!) and define a signal handler on SIGSEGV. For example, if the current ROM banks are written to, we may swap ROM or RAM banks.

Problem: Some memory regions smaller than 4096 bytes are partially r/o. If we make all memory write-protected, the segfaults may be too computationally intensive.

Another option would be to patch the jitted code after the first invokation to write to some auxiliary r/w page if it is not writing to r/o memory?

This is annoying since the GameBoy has no memory protection and plenty games execute code from RAM (e.g. DMA transfer during blanking, iirc).
