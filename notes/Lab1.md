# Lab1

## Introduction

- x86 assembly language
- **QEMU** x86 emulator
- PC's power-on **bootstrap** procedure
- **boot loader** for 6.828 kernel
- 6.828 kernel **JOS**



## Part 1: PC Bootstrap

### The PC's Physical Address Space

<img src="/Users/hiimswindler/Library/Application Support/typora-user-images/image-20240115143613777.png" alt="image-20240115143613777" style="zoom:75%;" />

#### 16-bit Intel 8088 processor

- physical address **(20 bits)** = segment address << 4 + offset **(Real Mode)**
- capable of adressing **1MB** of physical memory (from 0x00 ~ 0x100000)
- **640KB** area marked "Low Memory" was the only random-access memory (RAM)



### The ROM BIOS (IBM PC)

- BIOS in a PC is **hard-wired** to the physical memory (0xF0000 ~ 0xFFFFF)
- Always get control of the machine first after power-up or any system restart

- On processor reset, the (simulated) processor enters **real mode** and sets **CS** to 0xF000 and the **IP** to 0xFFF0
- Search for the bootable disk, and read the **boot loader** from the disk and **transfers** control to it.



## Part 2: The Boot Loader

#### Boot loader

- should be stored in the first sector (512B) of the bootable disk, which is called **boot sector**
- The 2nd sector onward holds the kernel code

##### boot.s

Set up **Protected Mode**

```assembly
#include <inc/mmu.h>

# Start the CPU: switch to 32-bit protected mode, jump into C.
# The BIOS loads this code from the first sector of the hard disk into
# memory at physical address 0x7c00 and starts executing in real mode
# with %cs=0 %ip=7c00.

.set PROT_MODE_CSEG, 0x8         # kernel code segment selector
.set PROT_MODE_DSEG, 0x10        # kernel data segment selector
.set CR0_PE_ON,      0x1         # protected mode enable flag

.globl start
start:
  .code16                     # Assemble for 16-bit mode
  cli                         # Disable interrupts
  cld                         # String operations increment

  # Set up the important data segment registers (DS, ES, SS).
  xorw    %ax,%ax             # Segment number zero
  movw    %ax,%ds             # -> Data Segment
  movw    %ax,%es             # -> Extra Segment
  movw    %ax,%ss             # -> Stack Segment

  # Enable A20:
  #   For backwards compatibility with the earliest PCs, physical
  #   address line 20 is tied low, so that addresses higher than
  #   1MB wrap around to zero by default.  This code undoes this.
  
  # using keyboard controller
seta20.1:
  inb     $0x64,%al               # read a byte from I/O port 0x64
  testb   $0x2,%al								# bitwise AND checks if busy flag is set
  jnz     seta20.1								# jump to seta20.1 if busy

  movb    $0xd1,%al               # 0xd1 -> port 0x64
  outb    %al,$0x64

seta20.2:
  inb     $0x64,%al               
  testb   $0x2,%al
  jnz     seta20.2							  # Wait for not busy

  movb    $0xdf,%al               # 0xdf -> port 0x60
  outb    %al,$0x60
  
  # long jump to next instruction, but in 32-bit code segment.
  # Switches processor into 32-bit mode.
  ljmp    $PROT_MODE_CSEG, $protcseg

  .code32                     # Assemble for 32-bit mode
protcseg:
  # Set up the protected-mode data segment registers
  movw    $PROT_MODE_DSEG, %ax    # Our data segment selector
  movw    %ax, %ds                # -> DS: Data Segment
  movw    %ax, %es                # -> ES: Extra Segment
  movw    %ax, %fs                # -> FS
  movw    %ax, %gs                # -> GS
  movw    %ax, %ss                # -> SS: Stack Segment
  
  # Set up the stack pointer (return address) and call into C.
  movl    $start, %esp
  call bootmain

  # If bootmain returns (it shouldn't), loop.
spin:
  jmp spin

# Bootstrap GDT (Global Descriptor Table)
.p2align 2                                # force 4 byte alignment
gdt:
  SEG_NULL				# null seg
  SEG(STA_X|STA_R, 0x0, 0xffffffff)	# code seg
  SEG(STA_W, 0x0, 0xffffffff)	        # data seg

# provide the size and address of GDT to the lgdt instruction
gdtdesc:
  .word   0x17                            # sizeof(gdt) - 1
  .long   gdt                             # address gdt

```

**Instruction explanation**

- **$0xd1:** Write Output Buffer Command. A command code that instructs the keyboard controller to expect a data byte (i.e., $0xdf) following this cmd
- **$0xdf:** Enable A20 Command. 

**The origin of A20**

A20 refers to the address line 20 in the x86 architecture. Addresses were represented using a combination of segment and offset values in early x86 system (e.g., 8086 had 20 address lines, ranging from A0 to A19). The A20 is a specific address line, and its activation allows for addressing memory beyond 1MB or 2^20B.



##### boot/main.c

Boot an ELF kernel image from the first IDE hard disk

```c
#include <inc/x86.h>
#include <inc/elf.h>

/**********************************************************************
 *
 * DISK LAYOUT
 *
 *  * The kernel image must be in ELF format.
 *
 * BOOT UP STEPS
 *  * when the CPU boots it loads the BIOS into memory and executes it
 *
 *  * the BIOS intializes devices, sets of the interrupt routines, and
 *    reads the first sector of the boot device(e.g., hard-drive)
 *    into memory and jumps to it.
 *
 *  * Assuming this boot loader is stored in the first sector of the
 *    hard-drive, this code takes over...
 *
 *  * control starts in boot.S -- which sets up protected mode,
 *    and a stack so C code then run, then calls bootmain()
 *
 *  * bootmain() in this file takes over, reads in the kernel and jumps to it.
 **********************************************************************/

#define SECTSIZE	512
#define ELFHDR		((struct Elf *) 0x10000) // scratch space

void readsect(void*, uint32_t);
void readseg(uint32_t, uint32_t, uint32_t);

void
bootmain(void)
{
  // ph: program header, eph: end of program header
	struct Proghdr *ph, *eph;

	// read 1st page (4KB) off disk
	readseg((uint32_t) ELFHDR, SECTSIZE*8, 0);

	// is this a valid ELF?
	if (ELFHDR->e_magic != ELF_MAGIC)
		goto bad;

	// load each program segment (ignores ph flags)
	// ELF segments: Text/Data/BSS/Others
	ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);
	eph = ph + ELFHDR->e_phnum;
	for (; ph < eph; ph++)
		// load kernel code
		readseg(ph->p_pa, ph->p_memsz, ph->p_offset);

	// call the entry point from the ELF header
	// note: does not return!
	// jump to kernel code, transfer control to kernel
	((void (*)(void)) (ELFHDR->e_entry))();

bad:
	outw(0x8A00, 0x8A00);
	outw(0x8A00, 0x8E00);
	while (1)
		/* do nothing */;
}

// Read 'count' bytes at 'offset' from kernel into physical address 'pa'.
// Might copy more than asked
void
readseg(uint32_t pa, uint32_t count, uint32_t offset)
{
	uint32_t end_pa;

	end_pa = pa + count;

	// round down to sector boundary (beginning of a sector)
	pa &= ~(SECTSIZE - 1);

	// translate from bytes to sectors, and kernel starts at sector 1
	offset = (offset / SECTSIZE) + 1;

	// If this is too slow, we could read lots of sectors at a time.
	// We'd write more to memory than asked, but it doesn't matter --
	// we load in increasing order.
	while (pa < end_pa) {
		// Since we haven't enabled paging yet and we're using
		// an identity segment mapping (see boot.S), we can
		// use physical addresses directly.  This won't be the
		// case once JOS enables the Memory Management Unit(MMU).
		readsect((uint8_t*) pa, offset);
		pa += SECTSIZE;
		offset++;
	}
}

void
waitdisk(void)
{
	// wait for disk reaady
	while ((inb(0x1F7) & 0xC0) != 0x40)
		/* do nothing */;
}

void
readsect(void *dst, uint32_t offset)
{
	// wait for disk to be ready
	waitdisk();

	outb(0x1F2, 1);		// count = 1
	outb(0x1F3, offset);
	outb(0x1F4, offset >> 8);
	outb(0x1F5, offset >> 16);
  // sets the highest three bits to indicate Logical Block Address(LBA)
	outb(0x1F6, (offset >> 24) | 0xE0); 
	outb(0x1F7, 0x20);	// cmd 0x20 - read sectors

	// wait for disk to be ready
	waitdisk();

	// read a sector
	insl(0x1F0, dst, SECTSIZE/4); // 128 words read to dst
}
```

**At what point does the processor start executing 32-bit code? What exactly causes the switch from 16- to 32-bit mode?**

```assembly
# in seta20.2
movb    $0xdf,%al               # 0xdf -> port 0x60
outb    %al,$0x60

# Switch from real to protected mode, using a bootstrap GDT
# and segment translation that makes virtual addresses 
# identical to their physical addresses, so that the 
# effective memory map does not change during the switch.
lgdt    gdtdesc
movl    %cr0, %eax
orl     $CR0_PE_ON, %eax    
movl    %eax, %cr0					# set the PE (Protection Enable) bit in CR0
```

**What is the *last* instruction of the boot loader executed, and what is the *first* instruction of the kernel it just loaded?**

```assembly
# boot loader
call *0x10018 # ((void (*)(void)) (ELFHDR->e_entry))();

# kernel
movw   $0x1234,0x472
```

***Where* is the first instruction of the kernel?**

```assembly
# 0x10000c
movw   $0x1234,0x472
```

**How does the boot loader decide how many sectors it must read in order to fetch the entire kernel from disk? Where does it find this information?**

The information is stored in EFL Header

```c
// ELF header structure
struct Elf {
    uint32_t e_magic;  // ELF identification
    // ... other fields
    uint16_t e_phoff;  // Program header table offset
    // ... other fields
};

// Program header structure
struct Proghdr {
    uint32_t p_type;   // Segment type (LOAD, dynamic linking information, etc.)
    uint32_t p_offset; // Offset it should be read
    uint32_t p_vaddr;  // Virtual address where it should be loaded
    uint32_t p_pa;     // Physical address where it should be loaded
    uint32_t p_filesz; // Size of the segment in the file, in bytes
    uint32_t p_memsz;  // Size of the segment in memory, in bytes
    // ... other fields
};
```

**Executable and Linkable File (ELF)**

![img](./img/ELF.png)

- `.text`: The program's executable instructions.
- `.rodata`: Read-only data, such as ASCII string constants produced by the C compiler. (We will not bother setting up the hardware to prohibit writing, however.)
- `.data`: The data section holds the program's initialized data, such as global variables declared with initializers like `int x = 5;`.
- `.bss`: The bss secotion immediately follows the `.data` section, and it reserves space for uninitialized global variables like `int x;`

Use `objdump -h` to check header information, look at the **VMA** (or link address) and the **LMA** (or load address) of each section. The LMA specifies the memory address at which this section should be loaded to the memory. The link address specifies the memory from which this section should be executed (entry point). 
