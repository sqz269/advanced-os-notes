# Extensibility, Safety and Performance in the SPIN Operating System

> Paper: https://dl.acm.org/doi/pdf/10.1145/224056.224077

## Problem Context

In theory, OS structure should support a wide range of use-cases and applications, while maintaining performance, and protection. However, some goals in OS structure have inherently contradicting objectives. For example, a scheduler can focus on throughput or responsiveness but not both (think about the overhead incured by frequenly switching processes). 

The overly ridgid design of OS kernels causes its performance characteristics/provided abstractions to mismatch the application's requirements. Although microkernels can provide some of the customization to the kernel, it's large overhead due to the need for frequent kernel-user context switches is a major limiting factor.

> For example, the implementations of disk buffering and paging algorithms found in modern operating systems can be inappropriate for database applications, resulting in poor performance

> The bottom line is that operating system services in many existing systems are either too slow or inappropilate. Current DBMSs usually provide their own and make little or no use of those offered by the operating system. It is important that future operating system designers become more sensitive to DBMS needs. ([Operating system support for database management](https://dl.acm.org/doi/10.1145/358699.358703))

## SPIN's Core idea

OS can be extensible, safe, and performant by leveraging language features (Modula-3). 

SPIN utilizes Modula-3's type safety system and moduls to allow applications to directly (and safely) inject code into the kernel and modify OS behavior to better match application's specific demands, without the overhead of IPC (microkernel) or traditional kernal hacks.

### Four Techniques

- Co-location
- Enforced Modularity
- Logical Protection Domains
- Dynamic Call Binding

## Approch/Architecture

### Modula-3

SPIN heavily utilizes Modula-3's interface system (It's very very similar to a interface you would see in a modern language like C#). In Modula-3, an interface defines the public API which can be accessed by another modules, while a module implements the functions exposed through an interface.

SPIN also relies on Modula-3's strong static type checking system, garbage collection, and strict pointer type safety to ensure the modules loaded into the kernel are safe.

### Capabilities

SPIN defines capabilities to control extensions access to certain kernel functions. Contrasting previous implementation of capabilities that requires hardware support, SPIN relies on Modula-3's interface to achieve capabilities. 


Concrete Example on how SPIN uses Module-3 (Directly from the paper)

```
INTERFACE Console; (* An interface. *)

TYPE T <: REFANY (* Read as “Console. T is opaque. ” *)
CONST InterfaceName = “ConsoleService”;
(* A global name *)

PROCEDURE Open:T;
(* Open returns a capability for the console. *)

PROCEDURE Write (t: T; msg: TEXT);
PROCEDURE Read(t: VAR msg: TEXT);
PROCEDURE Close (t: T);
END Console;
```
Notice that the only way to obtain a type `T` exposed by the console is to call `Open`, and since all of the Console procedures requires the type `T`, you have no choice but to call `Open` before calling any other procedures. And the OS can perform access checks when you call `Open`.

Acquiring & Using capabilities
```
MODULE Gatekeeper; (* A client *)
IMPORT Console;

VAR c: Console. T;  (* A capability for *)
                    (* the console device *)

PROCEDURE IntruderAlert () =
BEGIN

(* Obtain the capability, SPIN performs checks, etc.*)
c := Console.Open();


Console.Write(c, “Intruder Alert”) ;
Console.Close(c) ;

END IntruderAlert;

BEGIN

END GateKeeper;
```

#### Externalized Reference

The assumption of SPIN is that everything inside the kernel is written in Modula-3 and will be type-safe. But, issues can arise if Kernel wants to return a pointer to Userspace. Userspace programs can be written in non-type safe languages (C, asm, etc.), which can corrupt the pointer and thus break the assumption of memory safety. 

To mitigate this, instead of returning a raw pointer, the kernel return an index to the application, and keep tracks of a table that maps index with pointers per-process. The Kernel can just look up the table when it need the raw pointers.

This is very similar to what Linux do, think about `fopen`. When opening a file, Linux won't return a raw pointer to the Kernel's internal File struct, but instead gives you a file descriptor, and then store a map between your file descriptor to Kernel file pointer in a file descriptor table.

### Protection Domains

The Kernel can securely load a Modula-3 extensions using namespaces and dynamic linking. It's core capabilities involves the functions `create()`, `resolve()`, `combine()`

#### `create()`

A domain is created using the `create(interface name, domain)` operation, it will initialize the domain with the symbols exported by the extension. The Kernel nameserver will register the name and the exported symbols to it's nameserver and as lookups during resolve.

#### `resolve()`

`resolve(src domain, tgt domain)` looks up symbols unresolve in the target domain, and link the src domains symbol to the unresolved symbol.

#### `combine()`

`combine(domain1, domain2)` combines the symbols in both domains into one big domain. For example, the SpinPublic domain is a combination of all systems public interfaces to a single domain.

### Extension Model

SPIN's extensions can interact with the base system in various ways. The extension model allows SPIN extensions to monitor system activities, create hint for certain operations (e.g. page replacement), or entirely replaces a system service (e.g. schedulers). The communication between extensions and base systems are facilitated through events and handlers. Extensions implement and install event handlers by registering its handler through a central dispatcher. The default implementation can reject or accept the registration, if accepted, the dispatcher will install the event handler. In addition, the event handle can also specify guards to filter incoming events.

The events are dispatched through a central dispatcher and then routed to different event handlers. The default implementation (primary handlers) can constraint a handler to a specific execution style, sync/async, bounded time, or ordered/unordered. Depending on the return value the handlers, SPIN might need to aggregate results from handlers before returning a final result. (How?)

## Core Services

SPIN provides a set of core services that manages memory and CPU resources. The core services utilizes events between system and extensions.

These core services provides interfaces to hardware mechanisms, core services are considered trusted. Trusted services is needed because they need to access the underlying hardware which requires them to step outside of the protection model enforced by Modula-3.

### Memory Management

#### Phyiscal Address Service

#### Virtual Address Service

#### Translation Service

### Thread Management (Strand)

SPIN manages the CPU by mutiplexing the processor between multiple threads. However, SPIN does not explicitly define a thread model. Rather, it defines thread control events that extensions can utilize to implement their own thread model. In SPIN, each thread/runnable processes are refered as a "strand" which contains it's processor context (registers, etc.).

An extension (thread model) can interact with the schedule with two events, `Block` and `Unblock` and by raising these events, the thread model can signal the kernel a strand's execution state. And in response to those events, the scheduler can communicate with the threading model to save and restore the strand's state with `Checkpoint` and `Resume` events.

For example, an I/O driver can signal to the scheduler to de-schedule a strand that is in a blocking I/O call with `Block` and notify the scheduler when a strand is ready to run again with `Unblock`. While the scheduler can relinquish a strand from the processor with `Checkpoint` and give processor to the strand with `Resume`

## Thoughts


