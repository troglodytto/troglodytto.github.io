+++
date = '2026-02-03T00:52:24+05:30'
draft = false
title = 'Minimal MBR in Assembly'
slug = 'mbr'
+++

```asm {title="mbr.asm"}
; this is an assembler directive. org means origin, and 0x7c00 for the start address. 
; in essence, this tells the assembler to assume that the code is being loaded at address 0x7c00. 
; this is done to ensure that things that depend on knowing the current address upfront are correct.
; (e.g address offset calculations)
[org 0x7c00]

; since we are in real mode, we can't use more than 16 bits (due to compatibility reasons)
; so we make sure to emit instructions in 16 bits using 16 bit instruction encoding.
bits 16

start:
    ; we want to load into `es`, but you can't load into it directly
    ; hence we use a general purpose register `ax` to load the value
    ; and then load the value present in `ax` into `es`
    ; es <- ax <- 0xb800

    mov ax, 0xb800; load value into ax
    mov es, ax; then transfer the loaded value from ax, into es
    xor di, di; zeroing it (i.e reset/initialize to 0)

    ; at the end, es will contain the base (i.e 0xb800)
    ; and offset will be in di (right now set to zero)
    ; physical address will be calculated like so
    ; phys = ((es << 4) + di) & 0xfffff

    ; Why ds, si, es, di?
    ; intel in their x86 days wanted speed and efficiency
    ; so they hard wired operands to a bunch of instructions 
    ; such that whenever the CPU encounters those instructions, it can assume that 
    ; the source will usually be ds:si (ds base with si offset) 
    ; and when you write destination will always be es:di (es base with di offset)

    mov si, msg; set si to start of msg

    ; cx is a special register that is used as a counter
    ; it'll be helpful when we want to exit our loop later
    mov cx, msg_end - msg 

.write:
    ; think of lodsb as if it is doing this
    ; ah_register = ram[(ds_register << 4) + si_register] 
    ; si_register += 1
    lodsb
    
    mov ah, 0x07
    
    ; think of stosw as if it is doing this
    ; ram[(es_register << 4) + di_register] = ax_register
    ; di_register += 2
    stosw
    
    ; think of loop as if it is doing this
    ;
    ; cx_register -= 1
    ; if cx_register == 0
    ;     break
    ; else 
    ;     jump to label
    loop .write

.hang:
    jmp .hang

msg db "Hello world!"
msg_end:

times 510 - ($ - $$) db 0

dw 0xAA55
```

Let's go back to our [main post](../booting-into-our-kernel) and continue there.
