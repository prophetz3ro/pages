---
layout: page
title: ELF
permalink: /elf/
---
# ELF: Sections, Segments, Symbols, and Program Headers

## Sections

Sections organize data in an ELF file and are mainly used by the compiler, linker, and debugging tools.

Examples:

- `.text` — machine code
- `.data` — initialized global variables
- `.bss` — uninitialized global variables
- `.rodata` — read-only data
- `.symtab` — full symbol table
- `.dynsym` — dynamic symbol table

View sections:

```bash
readelf -S binary
```

---

## Symbols

Symbols represent named entities in a program.

Examples:

```c
int PI = 314;

int add(int a, int b) {
    return a + b;
}

int main() {
    return add(1, 2);
}
```

Possible symbols:

| Name | Type |
|--------|--------|
| PI | OBJECT |
| add | FUNC |
| main | FUNC |

A symbol entry contains metadata:

```text
Name      = main
Type      = FUNC
Value     = 0x1149
Size      = 42
Section   = .text
```

The actual machine code lives in `.text`; the symbol table only records information about it.

---

## .symtab vs .dynsym

### .symtab

Full symbol table.

Contains:

- Global functions
- Global variables
- Local/static symbols
- Section symbols
- Linker/debug information

Example:

```text
main
add
PI
static_helper
...
```

### .dynsym

Dynamic symbol table.

Contains symbols needed by the dynamic linker:

- Imported functions (`printf`, `malloc`, `free`)
- Exported symbols from shared libraries

Example:

```text
printf
malloc
free
```

View symbols:

```bash
readelf --syms binary
readelf --dyn-syms binary
```

---

## Segments

Segments describe how an executable should be loaded into memory.

Examples:

- `PT_LOAD`
- `PT_DYNAMIC`
- `PT_INTERP`
- `PT_PHDR`

View segments:

```bash
readelf -l binary
```

---

## Program Headers

Segments are defined by the Program Header Table.

```text
ELF
├── ELF Header
├── Program Header Table   <-- Segments
└── Section Header Table   <-- Sections
```

The kernel loads a program using the Program Header Table.

---

## Relationship Between Sections and Segments

Multiple sections are grouped into segments.

Example:

```text
PT_LOAD (R-X)
├── .init
├── .plt
├── .text
└── .rodata

PT_LOAD (RW-)
├── .data
└── .bss
```

Section-to-segment mapping:

```bash
readelf -l binary
```

Example output:

```text
Section to Segment mapping:
 Segment Sections...
  03     .init .plt .text .rodata
  04     .data .bss
```

---

## Important Rule

- Compiler/Linker → work with **Sections**
- Kernel/Dynamic Loader → work with **Segments (Program Headers)**

The kernel generally ignores section headers when loading an executable and uses only the program headers.