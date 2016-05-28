title: Introduction to Operation System
categories: 
    - Studying notes
    - Operating System
date: 2016-05-20
---
<img src="https://c1.staticflickr.com/8/7487/27172188552_99658bfdbd.jpg" width="500" height="313">
<!--more-->
----

## Operating System

Operating Systems Concerns:
* performance(time,space,energy)
* sharing and resource management
* failure tolerance
* security
* marketability

Hardware:
* disks(hard drives,optical drives)
* memory
* processors
* network(ethernet,modem)
* monitor
* keyboard
* mouse

	<img src="https://c2.staticflickr.com/8/7074/26663422633_1f7d9bbd0f.jpg" width="500" height="407">

OS Abstractions:
* disks->files(file system)
* memory->programs(processes)
* processors->threads of control
* network->communication
* monitor->windows,graphics
* keyboard->input
* mouse->locator

Issues with files abstraction:
* naming: device-independence
* allocating space on disk(permanent storage): organized for fast access, minimize waste
* shuffling data between disk and memory(high speed temporary storage)
* coping with crashes

address space(low address at the top):
* text(code) segment: executable code
* data segment: initialized global data
* bss segment: uninitialized global data
* dynamic(heap) segment
* stack segment: local variables, function arguments

	<img src="https://c8.staticflickr.com/8/7318/26663422743_fae4c97a3e.jpg" width="448" height="464">

memory map:
* part hardware, part OS
* each program thinks it has full address space

	<img src="https://c4.staticflickr.com/8/7564/26663422883_5860cbc56b.jpg" width="500" height="242">

## A simple OS

### OS Structure

Sixth-Edition Unix OS:
* 64KB of memory
* single file
* loaded into memory as OS boots
* monolithic OS

Processor modes:
* user mode: fewest privileges
* privileged mode(part of OS): most privileges

kernel: the portion of OS running in privileged mode

traps:invoking the kernel from user code, always elicit some response
* some for errors: divide by zero,segmentation fault,bus error
* some not: system calls,page fault

interrupts: invoking the kernel from external devices for a response
* handled independently of any user program(trap is handled as part of the program)
* response to interrupt has no direct effect on currently running program

	<img src="https://c6.staticflickr.com/8/7360/26663422933_22c37f762d.jpg" width="255" height="471">

software interrupt vs hardware interrupt:
* s:generated programmatically, h:generated by a device
* handling mechanisms are very similar

upcall: use signals invoke user program from kernel, executing signal handler
* terminate the program
* perform corrective action and continue with normal execution

x86 Processor registers:
* idx regs: EIP,ESP,EBP
* seg regs: CS(processor mode),SS
* gen regs: EAX
* other: flags(interrupt enabled)
* A0-A31,D0-D31,RD,WR,LOCK,INT

	<img src="https://c1.staticflickr.com/8/7385/26993591040_fe8c908328.jpg" width="500" height="320">

### Processes, Address Spaces, & Threads

abstraction of program execution:
* memory(address space)
* processors
* execution context: the state of a process

4 system calls for processes: fork(), exec(), wait(), exit()

turing machine:
* infinite tape divided into cells
* head
* state register
* finite table of instructions

objects is used for any data types:
* variables
* arrays
* dynamically create objects

### Managing Processes

create process: make a copy of a process(fork())
* parent process: return pid(16 bits) of child from fork
* child process: return 0 from fork

	<img src="https://c4.staticflickr.com/8/7036/27269255515_f5c89c8b8e.jpg" width="480" height="500">

process control block(PCB): kernel data structure
* pid
* address space description
* terminated children: points to linkedlist of dead child's PCB
* link: the process itself as a list node in a linkedlist
* return code: 8-bit long, communicate with parent process
* current state
* handles(open file descriptors): refer to an object managed by kernel

	<img src="https://c4.staticflickr.com/8/7178/27269255955_58f737c1e2.jpg" width="500" height="229">

thread control block(TCB): 
* stack pointer
* other registers
* state

```C
short pid;
if((pid=fork())==0){
	exit(n);
	/*copy least significant 8-bit of n into PCB*/
}else{
	int ReturnCode;
	while(pid!=wait(&ReturnCode));
	/*wait() is a blocking call, reaps dead child process*/
	return(ReturnCode);
	/*return to start up function, calling exit(ReturnCode)*/
}
```

when exit(), OS free up everything except PCB, go into a zombie state; after wait(), pid be reused and PCB be freed up

if parent call exit() while child in zombie state: init process inherits all zombie children, keep calling wait()

### Loading Program Into Processes

run a program:
* make a copy of a process: fork()
* replace child process with a new one(some stuff survives): exec()

	<img src="https://c2.staticflickr.com/8/7491/27269256145_0afb6517a2.jpg" width="500" height="353">
	<img src="https://c2.staticflickr.com/8/7157/27269256225_7167a4d314.jpg" width="457" height="480">
	<img src="https://c7.staticflickr.com/8/7616/26993592190_4fba5a9ee0.jpg" width="451" height="466">

address space composed of user portion and kernel portion, the same kernel for system calls spans across all user process
<img src="https://c3.staticflickr.com/8/7441/26993592290_ff43cba0b1.jpg" width="372" height="338">

### Files

files:
* fetching and storing data outside a process
* hierarchical naming structure
* part of a process's extended address space

directory system:
* shared by all processes running on a computer: each process have different view and root
* name space(handle) is outside the process:
 * user process provide file name to OS
 * OS return handle to access file along the entire path
 * user process use handle to read/write file

file abstraction:
* array of bytes
* be made larger by writing beyond current end
* named by path
* system calls on files are synchronous
* API: open(), read(), write(), close()

standard file descriptors:
* 0 is stdin, default map to keyboard
* 1 is stdout, default map to display
* 2 is stderr, default map to display

whenever a process request a new file descriptor, the lowest numbered file descriptor not already associated with an open file is selected

I/O redirection:
* \> in shell command redirect the output to given file
* < in shell command redirect the input from given file

file descriptor table(maintained by OS, in PCB, per process):
* file descriptor(handle, an index into an array) refer to its entry, 32 entries
* its entry points to system file table(system wide)

system file table(maintained by OS):
* reference count
* access mode
* file location: cursor
* inode pointer

inode table: inode form boundary between VFS and actual file system
<img src="https://c4.staticflickr.com/8/7418/27269256355_41c23bf0a7.jpg" width="500" height="277">

inter-process communication:
* if open file twice, two fd point to two extended address space, both point to the file, change second fd will influence first fd

	<img src="https://c8.staticflickr.com/8/7059/26663423983_7d9011584f.jpg" width="500" height="142">

* use dup() system call to share context information, ref of extended address space change to 2

	<img src="https://c6.staticflickr.com/8/7296/26663424013_773d79a21d.jpg" width="500" height="133">

* when fork(), child and parent process get separate file descriptor table but share extended address space

	<img src="https://c2.staticflickr.com/8/7436/26663424073_fd811defae.jpg" width="500" height="249">

* pipe():system call, create a pipe object in kernel, return two file descriptors refer to pipe. one for read, one for write

	<img src="https://c6.staticflickr.com/8/7668/26663424133_ec2725618f.jpg" width="500" height="304">

```C
int p[2];
pipe(p);
if(fork()==0){
	char buf[80];
	close(p[1]);
	while(read(p[0],buf,80)>0){}
	exit(0);
}else{
	char buf[80];
	close(p[0]);
	for(;;){
		write(p[1],buf,80);
	}
}
```

off_t lseek(int fd, off_t offset, int whence): move cursor position, whence could be SEEK_SET/SEEK_CUR/SEEK_END

file is represented as an inode in file system, directory is a kind of file, containing reference to other file/directories. 
* Directory map file name to inode number
* inode map inode number to sector on disk, done in actual file system

	<img src="https://c6.staticflickr.com/8/7460/26663422533_947f2b2869.jpg" width="482" height="416">
	<img src="https://c7.staticflickr.com/8/7127/26993590190_19e18babb6.jpg" width="287" height="251">

directory hierarchy deviation:
* hard link: reference to a file(not directory) in one directory also appear in another

	<img src="https://c5.staticflickr.com/8/7540/26993590140_f180185547.jpg" width="500" height="306">

* soft link(symbolic link): a special kind of file containing the name of another file or directory

	<img src="https://c1.staticflickr.com/8/7403/26993589920_5b492a8b69.jpg" width="500" height="315">

working directory: in kernel for each process, paths not starting from "/" start with the working directory

access protection:
* security principals:
 * user: owner of file
 * group: group owner of file
 * others: everyone else
* operations:
 * read: read a file or directory
 * write: write a file or directory
 * execute: have execute permission for a directory in order to follow a path through it
* rules for checking permission:
 * determine smallest class of principlas the requester belongs to
 * check for appropriate permissions with that class

	<img src="https://c3.staticflickr.com/8/7255/27172188722_af21bc0eb5.jpg" width="500" height="226">

setting file permission:
* int chmod(const char *path, mode_t mode):
 * only owner of file and superuser may change permission
 * have 9 possibilities for mode: S_IRUSR, S_IWUSR, S_IXUSR, S_IRGRP, S_IWGRP, S_IXGRP, S_IROTH, S_IWOTH, S_IXOTH
* permission=mode&~umask, umask used to turn off undesired permission