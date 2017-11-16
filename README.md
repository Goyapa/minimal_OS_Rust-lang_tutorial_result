# From "Writing an OS in Rust Philipp Oppermann's blog"

[Writing an OS in Rust Philipp Oppermann's blog](https://os.phil-opp.com/multiboot-kernel/)

"We can already assemble this file (which I called multiboot_header.asm) using nasm. It produces a flat binary by default, so the resulting file just contains our 24 bytes (in little endian if you work on a x86 machine):"
```
> nasm multiboot_header.asm
> hexdump -x multiboot_header
0000000    50d6    e852    0000    0000    0018    0000    af12    17ad
0000010    0000    0000    0008    0000
0000018
```
