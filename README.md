##                                PrivSep calling

####                 -finstrument-functions past and present      

I am instrumenting processes since more than 20y, mostly with very raw and basic
methods. On the plus side, you dont need to install *nodejs* on your machine
just to pop some memory contents off a process. 

I started building call graphs for *OpenSSH* somewhere in 2005. The aim was to
better understand the relations between the various `sshd` processes in advent
of `PrivilegeSeparation`. Call graphs by itself are little helpful for code reviews,
except for such cases where you struggle to find the IPC and privilege relations between
all the forked processes. The code that I wrote 12y ago is not working anymore today.
This document describes why and which hurdles you need to overcome in order to
produce a valuable call graph for an OpenSSH `sshd` session today.

#### Here we go

* `sshd` is not an `ET_EXEC` ELF file, but of type `ET_DYN`:

```
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Shared object file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x31337
...
```
  This means the kernel ELF loader will apply ASLR also for the `PT_LOAD`
  segments from the ELF base image itself, not just for the `DT_NEEDED` libraries which
  are loaded as dependencies by `ld.so`.
  This is what you get when building your binary with `-pie`. It is meant as a hardening
  measure against the [borrowed code chunks attack](https://events.ccc.de/congress/2005/fahrplan/attachments/554-Paper_NXx86-64BufferOverflowExploitsAndTheBorrowedNXcodeChunksExploitationTechnique.pdf) which some people wrongly call ROP.

* Therefore, static processing of libs and binaries is not sufficient to resolve addresses to symbol
  names. One has to check at runtime whats loaded and at which addresses before one can parse
  symbol tables.
* sshd is using a strange mix of processes, re-executions and threads inside these processes
  (let alone signal handlers) which must be synchronized to get a valuable
  instrumentation tracelog. This mix is due to the `PrivilegeSeparation` efforts of *OpenSSH*.
* Some of the synchronization primitives one would normally use in the C/C++ runtime environment
  may be using the `futex()` syscall, which is not an allowed syscall inside the `sshd` *seccomp* sandbox.
* `sshd` may be closing arbitrary file-descriptors at any time - most commonly when setting
  up new sessions - which is affecting instrumentation log-files. Re-opening files inside the *seccomp*
  sandbox is not allowed.
* In order to cluster the graph in a way it makes sense, one wants the tracelog in a single
  file and not scattered across directories and jails. As a bonus, inside the ascii call
  graph you can see the scheduler switching between processes.
 
You can find a complete `sshd` session inside this repo as dot and ascii files. Use `dotty` and
the bird view to navigate through the graph. Red nodes (functions) are running privileged (euid 0). Grey
is for unprivileged calls. Euid 496 is the *seccomp* sandbox user. You will see five sub-graphs,
each one denoting its own process. These are the offspring after the re-exec: *accepting* sshd,
its child *priv* sshd, sandboxed *net* sshd, *pam* sshd and the user-session sshd holding the pty.


