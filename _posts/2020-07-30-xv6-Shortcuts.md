---
title: Shortcuts for xv6-riscv
layout: post
comment: false
---
1. *cd /opt/riscv/bin*, after cd'ing run *./riscv64-unknown-linux-gnu-gdb ~/xv6-riscv-fall19/kernel/kernel*.
2. Sample gdbinit config,
add-auto-load-safe-path ~/xv6-riscv-fall19/.gdbinit

target remote localhost:26000

add-symbol-file ~/xv6-riscv-fall19/user/_lazytests_

Add symbols for whatever files you want to debug.

3. gdb shortcuts, 
b *filename:line_number* (but bp in file at file number)

c -> continue

n -> step over (next line)

s -> step into

4. Do *make qemu* for building without debugging. Do *make qemu-gdb* to be able to attach bp & debug.
