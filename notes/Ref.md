# CMD

**start QEMU**

```bash
# if not compile
cd lab
make

make qemu
```

**debug QEMU**

```bash
make qemu-gdb
make gdb
```

**breakpoints at critical entry**

```bash
b *0x7c00 # boot loader (boot.s)
b *0x7d15 # bootmain.c
b *0x7cdc # readseg
b *0x7c7c # readsect
```



# Material

[Links to notes etc](https://pdos.csail.mit.edu/6.828/2018/schedule.html)

[Lab Tools Guide](https://pdos.csail.mit.edu/6.828/2018/labguide.html)