---
layout: post
title: "Linux Kernel Modules and Volatility Notes"
date: 2025-09-04
categories: [forensics, linux, volatility]
tags: [kernel, rootkit, memory-forensics, volatility2]
permalink: /linux_kernel/
---

# What is a Linux Kernel Module?

A Linux kernel module (`.ko`) is code that can be loaded into the running kernel without rebooting.

Examples:

- Network drivers
- Filesystem drivers
- VirtualBox drivers
- Rootkits

List loaded modules:

```bash
lsmod
```

Load a module:

```bash
sudo insmod mymodule.ko
```

or

```bash
sudo modprobe mymodule
```

Unload:

```bash
sudo rmmod mymodule
```

# insmod vs modprobe

## insmod

Loads a specific `.ko` file.

```bash
sudo insmod mymodule.ko
```

## modprobe

Loads by module name and resolves dependencies.

```bash
sudo modprobe mymodule
```

# Module Parameters

Kernel modules can expose parameters.

Example:

```c
static char *key = "default";
module_param(key, charp, 0);
```

Load module with a custom value:

```bash
sudo modprobe mymodule key=secret123
```

Read runtime value:

```bash
cat /sys/module/mymodule/parameters/key
```

# Why Module Parameters Matter in Forensics

A parameter may not exist inside the module file itself.

Example:

```text
crc65_key=1337tibbartibbar
```

may have been supplied during module loading:

```bash
modprobe sysemptyrect crc65_key=1337tibbartibbar
```

Volatility can recover this runtime value from memory.

# Volatility 2 Linux Investigation

List loaded modules:

```bash
python2 vol.py -f dump.mem \
  --profile=<PROFILE> \
  linux_lsmod
```

Show module parameters:

```bash
python2 vol.py -f dump.mem \
  --profile=<PROFILE> \
  linux_lsmod -P
```

# Finding Hooked Syscalls

Check syscall hooks:

```bash
python2 vol.py -f dump.mem \
  --profile=<PROFILE> \
  linux_check_syscall
```

Example:

```text
sys_execve -> 0xffffffffc0a12470
```

Investigation workflow:

```text
Hook Address
      ↓
linux_lsmod
      ↓
Find module
      ↓
linux_moddump
      ↓
Ghidra / objdump
```

# Dumping a Kernel Module

List modules:

```bash
python2 vol.py -f dump.mem \
  --profile=<PROFILE> \
  linux_lsmod
```

Dump module:

```bash
python2 vol.py -f dump.mem \
  --profile=<PROFILE> \
  linux_moddump \
  -b <module_base> \
  -D modules/
```

Example:

```text
ffffffffc0a14020 sysemptyrect
```

# ELF Sections

## .text

Executable code.

## .rodata

Read-only strings and constants.

## .data

Initialized global variables.

## .bss

Zero-initialized global variables.

## .symtab

Symbol table. Maps addresses to functions and variables.

## .strtab

Symbol names used by `.symtab`.

## .dynsym

Dynamic symbols.

## .dynstr

Strings used by `.dynsym`.

## .shstrtab

Section names.

Example:

```text
.text
.data
.rodata
.symtab
.strtab
```

# Symbol Tables

View symbols:

```bash
nm module.ko
```

or

```bash
readelf -s module.ko
```

Example:

```text
0000000000001230 T check_password
0000000000001400 D secret_key
```

## Symbol Meanings

| Symbol | Meaning |
|----------|----------|
| T | Function |
| D | Initialized data |
| B | BSS variable |
| R | Read-only data |
| U | Imported symbol |

Stripped modules may show:

```text
ffffffffc0a12000 T
ffffffffc0a121d0 T
ffffffffc0a122f0 T
```

Function boundaries remain, but names are removed.

# Useful Reverse Engineering Commands

Show symbols:

```bash
nm -n module.ko
```

Show ELF sections:

```bash
readelf -S module.ko
```

Show symbol table:

```bash
readelf -s module.ko
```

Extract strings:

```bash
strings -a module.ko
```

Disassemble:

```bash
objdump -d -M intel module.ko
```

# Ghidra Workflow

1. Import module
2. Run Auto Analysis
3. Navigate to hooked syscall target
4. Open Decompiler
5. Review function logic
6. Search strings

Interesting keywords:

```text
key
secret
password
flag
auth
magic
```

# Key Lesson

A secret recovered with:

```bash
linux_lsmod -P
```

does not need to exist inside the module binary.

The value may have been supplied when the module was loaded:

```bash
modprobe sysemptyrect crc65_key=1337tibbartibbar
```

and later recovered from kernel memory during forensic analysis.