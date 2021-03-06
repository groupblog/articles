title: Basic Concepts of Operating System
categories: 
    - Studying Notes
    - Operating System
date: 2016-05-22
---
<img src="https://farm8.staticflickr.com/7446/27153713506_f5fa809db2_z.jpg" width="640" height="207">
<!--more-->
----

### Context Switching

OS switch applications transparently, each application don't know other applications' existance

execution context contains(stored in TCB):
* current state of the thread
* CPU registers: instruction pointer, stack pointer, base/frame pointer
* stack
* open files

in stack frame, a routine should store from high to low address:
* args, eip, ebp, saved registers, local variables
* esp points to the end of current stack frame, prepare for next stack frame
* eip contains caller's instruction pointer register, is return address
* ebp contains caller's base(frame) pointer register, link to the caller's ebp
* eax contains return value of a function
* args and eip are set by caller, others are set by callee

	<img src="https://farm8.staticflickr.com/7686/27153713456_878c2b8cc1_z.jpg" width="593" height="414">

there can be stuff between stack frames

routine code:
* set up stack frame
* push args
* pop args, get result
* set return value and restore frame

coroutine linkage: threads transfer control from one thread to another

<img src="https://farm8.staticflickr.com/7272/27153713396_00a5b5750d_z.jpg" width="640" height="378">

```C
void switch(thread_t *next_thread){
	CurrentThread->SP=SP;
	CurrentThread=next_thread;
	SP=CurrentThread->SP;
	return;
}
```

if TCB were user-space data structure, threads are switched without getting the kernel involved

one thread enters the switch() call, and a different thread leaves the switch() call

system calls: user code access kernel in a controlled manner:
* from user mode to priviledged mode, from user stack to kernel stack
* set things up
* traps(software interrupt) into kernel by executing a special machine instruction
* kernel handles the trap

	<img src="https://farm8.staticflickr.com/7246/27153713266_bd0ef7981c.jpg" width="294" height="220">
	<img src="https://farm8.staticflickr.com/7302/27091544582_e364ffd921.jpg" width="463" height="272">

process on trap:
* trap into kernel with interrupt disabled, set to kernel mode
* Hardware Abstractioin Layer save IP and SP in interrupt stack, HAL is hardware-dependent
* HAL set SP to point to kernel stack
* HAL set IP to interrupt handler
* pop IP and SP from interrupt stack, push onto kernel stack, re-enable interrupt
* on return from trap handler, disable interrupt, return to user process
* similar to hardware interrupt

signal generated by kernel, delivered to user process
interrupt generated by hardware, delivered to HAL then kernel

when interrupt occur:
* processor switch to an interrupt context
* previous context can be a thread context/another interrupt context
* when interrupt handler finish, processor resume original context

interrupt context stored in interrupt stack: borrowed kernel stack from the thread it is interrupting

<img src="https://farm8.staticflickr.com/7613/27153713136_8abf3ded99.jpg" width="500" height="346">

the handler of the most recent interrupt must run to completion, so interrupt service routine should do as little as possible:
* interrupt handler place most work on a queue, doing them in some other context later
* still need to do 2 things in interrupt handler: unblock a kernel thread, start the next I/O operation

a interrupt is pending if generated but masked, when unmasked, it's delievered

approaches to mask interrupt:
* bit vector/mask:
 * set a bit to enable/disable such interrupt
 * when an interrupt occur, the corresponding bit is set to block other interrupt of the same class
 * cleared when the handler return
* hierarchical interrupt levels:
 * processor set Interrupt Priority Level in hardware register
 * all interrupt with current or lower level are masked
 * kernel mask interrupt by setting IPL to a value
 * when an interrupt occur, IPL is set to that value
 * restore previous value on handler return

### Input/Output Architectures

memory-mapped I/O:
* device connect to controller, controller listen on the bus to determine request
* memory controller pass bus request to memory
* other controller process request, response to address

	<img src="https://farm8.staticflickr.com/7492/27153713036_eb343eebd6.jpg" width="500" height="148">

categories of devices:
* PIO(programmed I/O): read/write data in controller registers one byte at a time
* DMA(direct memory access):
 * controller perform I/O itself
 * process tell controller where to transfer data
 * controller take over and transfer data

device driver:
* a standard interface to OS
* code in it know how to talk to device
* OS treat I/O in a device-independent manner, using an array of function pointers

	<img src="https://farm8.staticflickr.com/7349/27153712966_05842fd774.jpg" width="500" height="218">

C++ polymorphism achieved using virtual base class, each device driver is a subclass, those class is often asynchronous

```C
class disk{
	public:
		virtual handle_t start_read(request_t)=0;
		virtual handle_t start_write(request_t)=0;
		virtual status_t wait(handle_t)=0;
		virtual status_t interrupt()=0;
};
```

### Dynamic Storage Allocation

dynamic storage allocation:
* best-fit
* first-fit
* buddy system
* slab allocation

first-fit works better compared to best-fit, best-fit leave many small memory regions, both have external gragmentation

fragmentation:
* internal fragmentation
* external fragmentation

first-fit data structure:
* a doubly-linked list of free blocks
* free block contains: 
 * prefix: in-use flag(4KB), size(total)(4KB), prev(4KB), next(4KB)
 * suffix: in-use flag(4KB), size(total)(4KB)
* in-use block contains:
 * prefix: in-use flag(4KB), size(total)(4KB)
 * suffix: in-use flag(4KB), size(total)(4KB)
* free block: check the block before it and after it, if neither of them are free, need to insert this block into the free list, else need to merge/coalesce the block with neighboring free block

	<img src="https://farm8.staticflickr.com/7326/27153712866_dd95368e49.jpg" width="500" height="299">
	<img src="https://farm8.staticflickr.com/7632/27153712796_8e486553c7.jpg" width="500" height="224">

first-fit algorithm: malloc() is O(n), free(ptr) is O(n)

buddy system: 
* blocks evenly divided into two as buddies with each other
* can only merge with buddy
* have internal fragmentation

	<img src="https://farm8.staticflickr.com/7022/27153712726_4acdbab3fe.jpg" width="500" height="311">

buddy system data structure:
* doubly-linked list free list indexed by k
* each block contains: in-use bit, size, next and prev links for free list

	<img src="https://farm8.staticflickr.com/7773/27091543002_b8646b07f3.jpg" width="500" height="107">

slab allocation:
* set up separate cache for each type of object in kernel
* contiguous sets of pages called slabs, allocated to hold objects
* slabs are initialized in advance, objects taking from the existing slabs in cache
* free object: simply marked as free

	<img src="https://farm8.staticflickr.com/7066/27091542612_e2332f15fd.jpg" width="500" height="439">

### Linking & Loading

location independent: everything in code can be accessed relative to frame pointer, use relative address

relocation:
* modify internal reference in memory depends on where module load(exec load program into memory)
* module requiring relocation is relocatable
* modifying module to resolve references is relocation
* program performing relocation is linker

function of linker:
* relocation
* symbol resolution

loader load program into memory, relocating loader can also do relocation

.c file compiled into .o file(relocatable modules), where have instruction for ld, then ld(linker&loader) modify .o file, copy into executable file

.o file contains data, bss and text sections, each section contains:
* global symbols
* undefined symbols
* instructions for relocation

	<img src="https://farm8.staticflickr.com/7295/27091542482_4889d5e125.jpg" width="500" height="322">

ld's work:
* lay out address space
* allocate memory in pages(4KB)
* main don't start from location 0, making first page inaccessible
* data segment start at a page boundary so that both segments have different access type(read-only/read-write)

	<img src="https://farm8.staticflickr.com/7346/27091542112_eec607f144.jpg" width="338" height="358">

library:
* a collection of .o files
* linker create libraries
* two types: static library, dynamic(shared) library

when compiling, order of libraries matter

problem for static library:
* execute file must contain everything for execution
* have duplicate code, taking up disk space and memory
* need a way to share code on disk and memory

	<img src="https://farm8.staticflickr.com/7126/26580986714_dd94faff5a.jpg" width="500" height="150">

solutions:
* limited sharing: relocate separately for each process
* prerelocation: relocate libraries ahead of time
* position-independent code: each process maintains a private table, pointed to by register. table contains address of shared routines

shared libraries:
* called Dynamic-Link Libraries in Windows, and shared objects in Unix
* don't need to load when program start up, can load on demand

disadvantages of shared libraries:
* have dependencies
* different versions of same library

linking and loading on linux with ELF:
* x86 ELF
* creating and using a shared library
* substitution
* shared library details
* versioning
* dynamic linking
* interpositioining

procedure of invoking program:
* given control to ld.so, the run-time linker
* ld does some initial set up of linkages
* call the actual program code
* may called on later to do some further loading and linking

versioning about libmyputs.so.1.0:
* real name: libmyputs.so.1.0
* soname: libmyputs.so.1
* linker name: libmyputs.so

### Booting

boot: load its OS into memory

solution: load a tiny OS into memory(bootstrap loader): 
* hard-wired to always run the code contained in its on-board read-only memory
* read bootstrap loader from floppy disk
* load OS from root directory of first file system on primary disk

floppy disk would handle:
* disk device
* on-disk file system
* need the right device driver
* need to know how disk is partitioned

new bootstrap loader, basic input-output system(BIOS):
* code stored in read-only memory(ROM)
* configuration data in non-volatile RAM(NVRAM), like CMOS
* provide three functions:
 * power-on self test(POST): know where to load boot program
 * load/transfer control to boot program
 * provide drivers for all device

on power-on, CPU execute BIOS(located in last 64KB of first 1GB of address space), start executing startup(the last 16 bytes of this region), jump to POST

POST:
* initialize hardware
* counts memory locations
* find a boot device
* load the Master Boot Record(MBR) from first sector of boot device

MBR contains magic number, partition table, boot program:
* one of partitions in partition table is active, that's boot point:
 * load first sector from it: contains volume boot program
 * pass control to it

	<img src="https://farm8.staticflickr.com/7746/26580986494_273a7baf45.jpg" width="315" height="472">

linux booting:
* provided by either lilo or grub
* set up stack, clear BSS, uncompress kernel, transfer control to it
* process 0 is created, set up initial page table, turn on address translation
* initialize rest of kernel, create init process, invoke the scheduler

BIOS device drivers:
* BIO provide drivers for all devices
* these drivers sat in low memory and provided minimal functionality