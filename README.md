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

## Building the Executable

To boot our executable later through GRUB, it should be an ELF executable. So we want nasm to create ELF object files instead of plain binaries. To do that, we simply pass the ‑f elf64 argument to it.

To create the ELF executable, we need to link the object files together. We use a custom linker script named linker.ld:

```
ENTRY(start)

SECTIONS {
    . = 1M;

    .boot :
    {
        /* ensure that the multiboot header is at the beginning */
        *(.multiboot_header)
    }

    .text :
    {
        *(.text)
    }
}

```
So let's create the ELF object files and link them using our new linker script:

```
> nasm -f elf64 multiboot_header.asm
> nasm -f elf64 boot.asm
> ld -n -o kernel.bin -T linker.ld multiboot_header.o boot.o
```
We can use objdump to print the sections of the generated executable and verify that the .boot section has a low file offset:

```
> objdump -h kernel.bin
kernel.bin:     file format elf64-x86-64

Sections:
Idx Name      Size      VMA               LMA               File off  Algn
  0 .boot     00000018  0000000000100000  0000000000100000  00000080  2**0
              CONTENTS, ALLOC, LOAD, READONLY, DATA
  1 .text     0000000b  0000000000100020  0000000000100020  000000a0  2**4
              CONTENTS, ALLOC, LOAD, READONLY, CODE
```
## Creating the ISO (loading from virual CD-ROM)

All PC BIOSes know how to boot from a CD-ROM, so we want to create a bootable CD-ROM image, containing our kernel and the GRUB bootloader's files, in a single file called an ISO. Make the following directory structure and copy the kernel.bin to the right place:

```
isofiles
└── boot
    ├── grub
    │   └── grub.cfg
    └── kernel.bin
```
The grub.cfg specifies the file name of our kernel and its Multiboot 2 compliance. It looks like this:
```
set timeout=0
set default=0

menuentry "my os" {
    multiboot2 /boot/kernel.bin
    boot
}
```

Now we can create a bootable image using the command:
```
grub-mkrescue -o os.iso isofiles
```
### Booting

Now it's time to boot our OS. We will use QEMU:
```
qemu-system-x86_64 -cdrom os.iso
```

## Build Automation

```
…
├── Makefile
└── src
    └── arch
        └── x86_64
            ├── multiboot_header.asm
            ├── boot.asm
            ├── linker.ld
            └── grub.cfg
```

The Makefile looks like this (indented with tabs instead of spaces):

```
arch ?= x86_64
kernel := build/kernel-$(arch).bin
iso := build/os-$(arch).iso

linker_script := src/arch/$(arch)/linker.ld
grub_cfg := src/arch/$(arch)/grub.cfg
assembly_source_files := $(wildcard src/arch/$(arch)/*.asm)
assembly_object_files := $(patsubst src/arch/$(arch)/%.asm, \
    build/arch/$(arch)/%.o, $(assembly_source_files))

.PHONY: all clean run iso

all: $(kernel)

clean:
    @rm -r build

run: $(iso)
    @qemu-system-x86_64 -cdrom $(iso)

iso: $(iso)

$(iso): $(kernel) $(grub_cfg)
    @mkdir -p build/isofiles/boot/grub
    @cp $(kernel) build/isofiles/boot/kernel.bin
    @cp $(grub_cfg) build/isofiles/boot/grub
    @grub-mkrescue -o $(iso) build/isofiles 2> /dev/null
    @rm -r build/isofiles

$(kernel): $(assembly_object_files) $(linker_script)
    @ld -n -T $(linker_script) -o $(kernel) $(assembly_object_files)

# compile assembly files
build/arch/$(arch)/%.o: src/arch/$(arch)/%.asm
    @mkdir -p $(shell dirname $@)
    @nasm -felf64 $< -o $@
```

```
make
make iso
make run

```

