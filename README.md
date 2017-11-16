# From "Writing an OS in Rust Philipp Oppermann's blog"



Source text is shorten for my own use and recapture, refer to [Writing an OS in Rust Philipp Oppermann's blog](https://os.phil-opp.com/multiboot-kernel/).

## Multiboot

To indicate our Multiboot 2 support to the bootloader, our kernel must start with a Multiboot Header:
```
section .multiboot_header
header_start:
    dd 0xe85250d6                ; magic number (multiboot 2)
    dd 0                         ; architecture 0 (protected mode i386)
    dd header_end - header_start ; header length
    ; checksum
    dd 0x100000000 - (0xe85250d6 + 0 + (header_end - header_start))

    ; insert optional multiboot tags here

    ; required end tag
    dw 0    ; type
    dw 0    ; flags
    dd 8    ; size
header_end:
```


We can already assemble this file (which I called multiboot_header.asm) using nasm. It produces a flat binary by default, so the resulting file just contains our 24 bytes (in little endian if you work on a x86 machine):


```
> nasm multiboot_header.asm
> hexdump -x multiboot_header
0000000    50d6    e852    0000    0000    0018    0000    af12    17ad
0000010    0000    0000    0008    0000
0000018
```

## The Boot Code

To boot our kernel, we must add some code that the bootloader can call. Let's create a file named boot.asm:
```
global start

section .text
bits 32
start:
    ; print `OK` to screen
    mov dword [0xb8000], 0x2f4b2f4f
    hlt
```

Through assembling, viewing and disassembling we can see the CPU Opcodes in action:

```
> nasm boot.asm
> hexdump -x boot
0000000    05c7    8000    000b    2f4b    2f4f    00f4
000000b
> ndisasm -b 32 boot
00000000  C70500800B004B2F  mov dword [dword 0xb8000],0x2f4b2f4f
         -4F2F
0000000A  F4                hlt
```

