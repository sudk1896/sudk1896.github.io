---
title: Shortcuts for xv6-riscv
layout: post
comment: false
---
- *cd /opt/riscv/bin*, after cd'ing run *./riscv64-unknown-linux-gnu-gdb ~/xv6-riscv-fall19/kernel/kernel*.
- Sample gdbinit config,
`
add-auto-load-safe-path ~/xv6-riscv-fall19/.gdbinit

target remote localhost:26000

add-symbol-file ~/xv6-riscv-fall19/user/_lazytests_

Add symbols for whatever files you want to debug.
`

- gdb shortcuts,
` 
b *filename:line_number* (but bp in file at file number)

c -> continue

n -> step over (next line)

s -> step into
`

- Do *make qemu* for building without debugging. Do *make qemu-gdb* to be able to attach bp & debug.
