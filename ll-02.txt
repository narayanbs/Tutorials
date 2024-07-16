##### Overview
There are several steps executed while spawning a process in Linux. If a program depends on shared libraries, a dynamic linker is also loaded to the memory in addition to the executable binary program.

Before learning about the dynamic linker, we’ll first briefly discuss the execution of a program to understand where the dynamic linker takes the stage.

When we start a process from the command line, the shell first calls the execve() system call to execute the program. It opens the executable file, and after some preparation, loads the executable file into memory.

If the executable file has shared library dependencies, an interpreter is also loaded and run to assemble the dependent shared libraries. This interpreter is the dynamic linker.

Once the dynamic linker finishes its job, the control passes to the user program.

ELF (Executable and Linkable Format) is the standard binary file format in Linux. It is also the format used by Object files (.o) and 
shared libraries (.so) files.  Note that static libraries for ex: libc.a is just an archive of object files. It is not an elf file. 

An ELF file for an executable always has a part called the Program Header Table. This part provides information that is necessary while running the executable.

One of the entries in the Program Header Table that is important while running the executable is the PT_INTERP entry. This entry specifies the run-time or dynamic linker needed to assemble the complete program before running.

The dynamic linker performs several tasks:

- Locating the necessary shared libraries
- Loading the shared libraries into memory
- Resolving the program’s undefined symbols using the symbols in shared libraries
- Assembling the program so that the process can call the functions in the shared libraries at run-time
 
In other words, the interpreter specified by the PT_INTERP entry in the Program Header Table manages the shared libraries at run-time on behalf of the executable.

The dynamic linker is by default set to /usr/lib64/ld-linux-x86-64.so.2 by gcc. We can check its presence using the readelf command:
```
$ readelf -l /usr/bin/ls | head -20
```


```
Elf file type is DYN (Shared object file)
Entry point 0x6b10`
```
There are 13 program headers, starting at offset 64
```
Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000000040 0x0000000000000040
                 0x00000000000002d8 0x00000000000002d8  R      0x8
  INTERP         0x0000000000000318 0x0000000000000318 0x0000000000000318
                 0x000000000000001c 0x000000000000001c  R      0x1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000003510 0x0000000000003510  R      0x1000
  LOAD           0x0000000000004000 0x0000000000004000 0x0000000000004000
                 0x0000000000013111 0x0000000000013111  R E    0x1000
  LOAD           0x0000000000018000 0x0000000000018000 0x0000000000018000
                 0x0000000000007530 0x0000000000007530  R      0x1000
  LOAD           0x000000000001ff70 0x0000000000020f70 0x0000000000020f70
```

We use the head command to have a shorter output. The -l option of readelf lists the program headers in an ELF file. We list the program headers of /usr/bin/ls as an example. As we see in the INTERP entry of the Program Headers, the executable requires the dynamic linker `/lib64/ld-linux-x86-64.so.2` as the interpreter.

Therefore, the dynamic linker is run indirectly when we run an executable with shared library dependencies. However, it’s also possible to run a dynamically linked program directly by calling the dynamic linker. We’ll see an example in the next section.

The ldd command also lists the dynamic linker: (ldd just displays the shared object dependencies of an executable)
```
$ ldd /usr/bin/ls
	linux-vdso.so.1 (0x00007fff54f89000)
	libselinux.so.1 => /lib64/libselinux.so.1 (0x00007f1000b17000)
	libcap.so.2 => /lib64/libcap.so.2 (0x00007f1000b0d000)
	libc.so.6 => /lib64/libc.so.6 (0x00007f1000903000)
	libpcre2-8.so.0 => /lib64/libpcre2-8.so.0 (0x00007f100086c000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f1000b74000)
```

The dynamic linker is the last entry in the output.

3. An Example
In this section, we’ll experiment with the dynamic linker using an example.

3.1. Example Code
We’ll use the following C program in our example, hello.c:
```
#include <stdio.h>

void main(int ac, char **av)
{
    printf("Hello %s\n", av[1]);
}
```

The program just greets the name passed to it as a parameter using the printf(“Hello %s\n”, av[1]) statement.

Let’s build the program using gcc to generate the executable binary hello:

```
$ gcc -o hello hello.c
$ ls -l hello
-rwxrwxr-x 1 centos centos 25680 Aug 27 05:17 hello
```

Having built the executable, let’s run it:
```

$ ./hello Baeldung
Hello Baeldung
```
The executable runs as expected.

3.2. Using the Dynamic Linker Explicitly

Now, let’s run hello using the dynamic linker, /lib64/ld-linux-x86-64.so.2:
```
$ /lib64/ld-linux-x86-64.so.2 ./hello Baeldung
Hello Baeldung
```

The executable runs as expected again. We just pass the executable together with its parameters to the dynamic linker.

We can run a binary file using the dynamic linker even if the binary file doesn’t have execute rights:
```
$ chmod -x hello
$ ls -l hello
-rw-rw-r-- 1 centos centos 25680 Aug 27 05:17 hello
$ /lib64/ld-linux-x86-64.so.2 ./hello Baeldung
Hello Baeldung
```
As the output shows, after removing the execute rights of the binary hello using chmod -x hello, we’re still able to run the application using the dynamic linker.

##### How does the above work???
When we do /lib64/ld-linux-x86-64.so.2 ./hello Baeldung, the check for execute permission is done by execve on /lib64/ld-linux-x86-64.so.2  and not on ./hello

The dynamic linker can be run on its own . try
```
$ /lib64/ld-linux-x86-64.so.2
/lib64/ld-linux-x86-64.so.2: Missing program name
Try '/lib64/ld-linux-x86-64.so.2 --help' for more information

$ /lib64/ld-linux-x86-64.so.2 --help
Usage: /lib64/ld-linux-x86-64.so.2 [OPTION]... EXECUTABLE-FILE [ARGS-FOR-PROGRAM...]
You have invoked 'ld.so', the program interpreter for dynamically-linked
ELF programs.  Usually, the program interpreter is invoked automatically
when a dynamically-linked executable is started.

You may invoke the program interpreter program directly from the command
line to load and run an ELF executable file; this is like executing that
file itself, but always uses the program interpreter you invoked,
instead of the program interpreter specified in the executable file you
run. 

```

##### How does a shared library act as an executable???
That library has a `main()` function or equivalent entry point, and was compiled in such a way that it is useful both as an executable and as a shared object.

Here is how we may do it
First, source for our example library, `test.c`:

```c
#include <stdio.h>                  

void sayHello (char *tag) {         
    printf("%s: Hello!\n", tag);    
}                                   

int main (int argc, char *argv[]) { 
    sayHello(argv[0]);              
    return 0;                       
}                   
```

Compile that:

```
gcc -fPIC -pie -o libtest.so test.c -Wl,-E
```

Here, we are compiling a shared library (`-fPIC`), but telling the linker that it's a regular executable (`-pie`), and to make its symbol table exportable (`-Wl,-E`), such that it can be usefully linked against.

And, although `file` will say it's a shared object, it does work as an executable:

```
> ./libtest.so 
./libtest.so: Hello!
```

Now we need to see if it can really be dynamically linked. An example program, `program.c`:

```c
#include <stdio.h>

extern void sayHello (char*);

int main (int argc, char *argv[]) {
    puts("Test program.");
    sayHello(argv[0]);
    return 0;
}
```

Using `extern` saves us having to create a header. Now compile that:

```
gcc program.c -L. -ltest
```

Before we can execute it, we need to add the path of `libtest.so` for the dynamic loader:

```
export LD_LIBRARY_PATH=./
```

Now:

```
> ./a.out
Test program.
./a.out: Hello!
```

And `ldd a.out` will show the linkage to `libtest.so`.


3.3. Setting the Dynamic Linker Explicitly
As we’ve already seen, gcc sets /lib64/ld-linux-x86-64.so.2 as the dynamic linker by default. However, it’s possible to set the dynamic linker explicitly during linkage.

Firstly, let’s copy /lib64/ld-linux-x86-64.so.2 to /tmp with the name my_ld.so:
```
$ cp /lib64/ld-linux-x86-64.so.2 /tmp/my_ld.so
```

We’ll specify /tmp/my_ld.so as the dynamic linker while building hello:
```

$ gcc -o hello hello.c -Wl,-I/tmp/my_ld.so
```

The -Wl option of gcc lets us pass linker options. The -Wl,-I/tmp/my_ld.so in our case specifies /tmp/my_ld.so to be used as the dynamic linker. Let’s check the PT_INTERP entry in the Program Header Table using readelf:
```

$ readelf -l ./hello | grep -B2 interpreter
  INTERP         0x0000000000000318 0x0000000000400318 0x0000000000400318
                 0x000000000000000e 0x000000000000000e  R      0x1
      [Requesting program interpreter: /tmp/my_ld.so]
```
We use grep to refine the output of readelf. As is apparent from the output, the dynamic linker is now /tmp/my_ld.so. Let’s run hello using /tmp/my_ld.so:
```
$ /tmp/my_ld.so ./hello Baeldung
Hello Baeldung
```
The program runs as expected. Therefore, we’re successful in changing the default dynamic linker.


Additional:
------------
The following is taken from https://lwn.net/Articles/631631/  (how programs get run: Elf binaries )
*** 
So far we've assumed the program being executed is statically linked and skipped over steps that would be triggered by the presence of a PT_INTERP entry in the ELF program header. However, most programs are dynamically linked, meaning that required [shared libraries](http://lurklurk.org/linkers/linkers.html#sharedlibs) have to be located and linked at run-time. This is performed by the [runtime linker](http://man7.org/linux/man-pages/man8/ld.so.8.html) (typically something like /lib64/ld-linux-x86-64.so.2), and the identity of this linker is specified by the PT_INTERP program header entry.

To cope with a runtime linker, the ELF handler first [reads the ELF interpreter file name](https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_elf.c#L660) into scratch space, then [opens the executable file](https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_elf.c#L673) with [open_exec()](https://elixir.bootlin.com/linux/v3.18/source/fs/exec.c#L786). The first 128 bytes of the file are read into the bprm->buf scratch area (bprm is a struct),   replacing the contents of the original program file and allowing [access](https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_elf.c#L694) to the ELF header of the interpreter program — which [must](https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_elf.c#L711) therefore be an ELF binary itself, rather than any other format.

After the program code has been loaded into memory as described previously, the ELF handler also [loads the ELF interpreter program](https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_elf.c#L890) into memory with [load_elf_interp()](https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_elf.c#L395). This process is similar to the process of loading the original program: the code [checks the format information in the ELF header](https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_elf.c#L408), [reads in the ELF program header](https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_elf.c#L427), [maps](https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_elf.c#L470) all of the PT_LOAD segments from the file into the new program's memory, and [leaves room](https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_elf.c#L518) for the interpreter's BSS segment.

The execution start address for the program is also [set to be the entry point](https://elixir.bootlin.com/linux/v3.18/source/fs/binfmt_elf.c#L900) of the interpreter, rather than that of the program itself. When the execve() system call completes, execution then begins with the ELF interpreter, which takes care of satisfying the linkage requirements of the program from user space — finding and loading the shared libraries that the program depends on, and resolving the program's undefined symbols to the correct definitions in those libraries. Once this linkage process is done (which relies on a much deeper understanding of the ELF format than the kernel has), the interpreter can start the execution of the new program itself, [at the address previously recorded](https://sourceware.org/git/?p=glibc.git;a=blob;f=elf/dl-sysdep.c;h=65a90469c6bdd12cba03c5a21a283971db39868d;hb=c758a6861537815c759cba2018a3b1abb1943842#l129) in the AT_ENTRY auxiliary value.

*** 
