+++
date = '2026-02-03T00:52:24+05:30'
draft = false
title = 'Minimal MBR in Assembly'
slug = 'mbr'
+++

After the BIOS firmware is done with all its initialization routines, and validations, it loads code from the first sector on disk into memory address `0x7c00`.

The instruction pointer is also set to `0x7c00`. Meaning, the CPU starts executing code at that address.

In this very rudimentary MBR section, we'll try to write a string ("Hello world!") to the screen via the [VGA Text Buffer](https://wiki.osdev.org/Printing_To_Screen)

## VGA Text Buffer and MMIO
The [VGA Text Buffer](https://wiki.osdev.org/Printing_To_Screen) is a memory mapped I/O buffer which means the firmware reserves a memory address and map it to a given I/O device, instead of physical RAM on your computer.

When you access this address, the CPU takes that request and instead of treating it like any other memory operation, it routes that request to the given I/O devices.

In case of VGA Text Buffer, the physical memory address `0xB8000` is reserved by the firmware and is mapped to the Video Graphics Array.

When reading, it doesn't really return anything meaningful, so the CPU just receives a bunch of zeros. However, writing to that address gives us a powerful tool to display text on the screen.

When you write data to this buffer in a specific format, text characters show up on the screen. (we'll discuss that format briefly in a bit)

Thus, our goal is to write bytes to this address (and a few subsequent addresses), so that our desired output shows up on the screen.

Keep in mind that this only works with BIOS. UEFI doesn't natively support it.

## VGA Text Buffer cell format

The format is relatively simple. The VGA Text buffer is a linear buffer with 2000 cells arranged in a 80x25 character grid.

Each character is made up of two bytes. 

The least significant byte represents the ASCII character code of the character. Well.. not exactly ASCII. It uses a special set of characters called the [Code Page 437](https://en.wikipedia.org/wiki/Code_page_437). This set is mostly compatible with ASCII but it adds a bunch of additional glyphs above 128 characters. Since there are only 8 bits available to us, we get a total of 256 characters to work with, including 128 ASCII characters and 128 additional glyphs.

The most significant byte defines how the character looks i.e, it controls its foreground color, its background color, and blinking behavior. 

The most significant nybble of the byte (i.e first four bits from left) represents the background color.

and the least significant nybble (i.e last four bits from the left) represents the foreground color. 

The most significant bit (i.e the first bit from left) is also repurposed as a blinking bit which makes the character blink at 1-2 Hz (Although it needs to be configured via the Attribute Control register).

The color table itself looks like this (**Thank you [OS Dev Wiki](https://wiki.osdev.org/Text_UI) for providing this**):
![VGA Text Color Palette](/vga-text-buffer-color-palette.png)

In essence a single VGA Text Buffer cell looks like this.

![VGA Character Cell](/vga-character-cell.svg)

## Writing to MMIO
In order to see our desired text on the screen, we will write bytes to memory starting at the VGA MMIO address `0xb8000`

This is how the memory buffer should look like after we're done with it.

---
| **Address (Base)**  | **Character** | **Background Color (Base + 8 bits)** | **Foreground Color (Base + 8 bits + 4 bits)** |
| ------------------- | ------------- | ------------------------------------ | --------------------------------------------- |
| 0xb8000             | H             | Black                                | Green                                         |
| 0xb8002             | e             | Black                                | Green                                         |
| 0xb8004             | l             | Black                                | Green                                         |
| 0xb8006             | l             | Black                                | Green                                         |
| 0xb8008             | o             | Black                                | Green                                         |
| 0xb800a             | (Space)       | Black                                | Green                                         |
| 0xb800c             | W             | Black                                | Green                                         |
| 0xb800e             | o             | Black                                | Green                                         |
| 0xb8010             | r             | Black                                | Green                                         |
| 0xb8012             | l             | Black                                | Green                                         |
| 0xb8014             | d             | Black                                | Green                                         |
| 0xb8016             | !             | Black                                | Green                                         |
| 0xb8017..END        | (Space)       | Black                                | Green                                         |
---

In order to achieve this, we'd need to do the following (pseudo code)

```asm
// 80x25 cells is the standard VGA Text buffer resolution
width = 80
height = 25

// VGA text buffer is memory mapped, starting at physical address `0xb8000`
vga_buffer_start = 0xb8000
cell_count = width * height
vga_buffer_end = vga_buffer_start + (cell_count * 2) // each cell is 2 bytes in length

background = 0x0 // 0 is Black (refer to the color palette table above)
foreground = 0x2 // 2 is Green (refer to the color palette table above) 

// VGA text mode packs the foreground and background colors into a single byte. 
// the background uses the upper four bits, so we shift it left by four positions. 
// this aligns it correctly and guarantees the lower four bits are empty (set to zero) for the foreground color.
normalized_background = background << 4

attribute = normalized_background | foreground
characters = ['H', 'e', 'l', 'l', 'o', ' ', 'W', 'o', 'r', 'l', 'd', '!']

address = vga_buffer_start

// Write Text
for character in characters {
    memory[address] = character
    memory[address + 1] = attribute
    address += 2
}

// Cleanup Rest (set everything to empty space)
while address < vga_buffer_end {
    memory[address] = ' '
    memory[address + 1] = attribute
    address += 2
}
```

```asm {title="mbr.asm"}
[org 0x7c00]
bits 16

FG equ 0x2 ; Green
BG equ 0x0 ; Black

FOREGROUND equ FG & 0x0F
BACKGROUND equ (BG << 4) & 0xF0

ATTRIBUTE equ FOREGROUND | BACKGROUND

VGA_BUFFER_WIDTH equ 80
VGA_BUFFER_HEIGHT equ 25

start:
    mov ax, 0xb800
    mov es, ax
    xor di, di
    mov si, msg

.message_length_counter: 
    mov cx, msg_len

.write:
    lodsb
    mov ah, ATTRIBUTE
    stosw
    loop .write

.clear_counter: 
    mov cx, ((VGA_BUFFER_WIDTH * VGA_BUFFER_HEIGHT) - msg_len)

.clear:
    mov ah, ATTRIBUTE
    mov al, ' '
    stosw
    loop .clear

.hang:
    hlt
    jmp .hang

msg db "Hello world!"
msg_len equ $ - msg

times 510 - ($ - $$) db 0

dw 0xAA55
```

Let's go back to our [main post](../booting-into-our-kernel/#a-bootloader) and continue there.
