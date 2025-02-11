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

## Approch

### Modula-3

### Co-location

### Enforced Modularity (Isolation)

### Logical Protection Domains

### Dynamic Call Binding

## SPIN Detailed Architecture

### Protection Model

#### Capabilities

## Thoughts


