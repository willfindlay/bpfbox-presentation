# Slide 0: Title Slide

<!--
- I'm William Findlay from Carleton University and today I'll be talking about
  bpfbox, a new process conferment mechanism we have implemented for Linux using
  a Linux kernel technology called eBPF.
-->

- I'll be presenting bpfbox, a simple yet precise process confinement mechanism
  leveraging eBPF

- bpfbox is joint work with Anil Somayaji and David Barrera, also of Carleton
  University.

# Slide 1: bpfbox at a Glance

- At a glance, bpfbox represents a new approach to process confinement under the
  Linux kernel, enabled by eBPF, a kernel extension and observability framework.

- Users write per-application policy in a simple policy language and policy is
  then loaded and enforced in the kernel using eBPF programs attached to LSM
  hooks.

- Our eBPF-based implementation enables the integration of both userspace and
  kernelspace system state with LSM layer enforcement, something which no
  existing process confinement mechanism can do.


# Slide 2: Motivation

- Our motivation for creating an eBPF-based process confinement mechanism was
  informed by the dual insight that there is room for improvement in the
  existing process confinement landscape and that eBPF presents new
  opportunities for improving operating system security

- Higher level process confinement mechanisms on Linux such as Snap and Docker
  are generally made up of a combination of multiple lower level techniques. The
  idea is that these techniques taken together can produce a complete solution.
  Often, the policy is defined by writing high level package manifests for the
  application which are then translated into the underlying policy mechanisms.
  Unfortunately this often results in policy that is easy to write but difficult
  to audit and often overly permissive.

- Lower level process confinement mechanisms such as seccomp-bpf and mandatory
  access control frameworks such as SELinux, AppArmor, and TOMOYO are often
  cited as being difficult to use. These frameworks are designed to be used by
  security experts and are generally inaccessible to end users.

- We set out to see if we could do any better by building something new.

# Slide 3: eBPF Changes the Game

- eBPF changes the way we look at operating system security by offering
  a fine-grained system introspection framework that can be used to
  integrate cross-layer system state with policy enforcement.

- Further, it enables rapid prototyping and safe deployment of kernel
  extensions, which can be integrated into a running kernel without the need to
  reboot.

- This culminates in an opportunity to rethink process confinement from the
  ground up.


# Slide 4: eBPF in the Beginning

- eBPF stands for Extended Berkeley Packet Filter... But it actually has very
  little to do with Berkeley, packets, or filtering nowadays. In the beginning,
  BPF was designed to provide efficient and safe packet filtering capabilities
  for Unix. The name is now preserved for historical reasons, but packet
  filtering is a small subset of modern eBPF's capabilities.

- So then what _is_ eBPF?
- eBPF is a major re-write of the Linux BPF engine merged into the mainline
  kernel in 2014 by Alexei Starovoitov and Daniel Borkmann. It allows privileged
  users to load programs into the kernel that are then attached to specific
  system events. The original point of eBPF was offering fine-grained,
  cross-layer system introspection. BPF programs could be written to observe
  userspace and kernelspace, as well as the interfaces between the two.

# Slide 5: What Can eBPF Do?

- eBPF is capable of hooking into nearly every aspect of the running system,
  including:
    - The full networking stack
    - Userspace functions
    - System calls and kernelspace functions
    - System hardware and stack traces
    - And, more recently, LSM hooks for security monitoring

# Slide 6: How eBPF Works

- eBPF works by compiling code written in the C programming language into
  a special BPF bytecode with its own eBPF instruction set. From there, the
  user can select one of many BPF front-end frameworks to load the BPF program
  into the kernel via the BPF system call. BPF frameworks are offered as libraries
  for many major programming languages, including Python, Go, C/C++, and Rust.

- Once loaded, an in-kernel verifier analyzes the code for safety before either
  accepting or rejecting it. Once a BPF program is accepted, it is attached to
  one or more system events, during which it is executed via just-in-time
  compilation to the native instruction set, such as x86.

- BPF programs can store information about the system state in special map
  data structures which can also be accessed from userspace.

# Slide 7: eBPF in 2020

- In 2020, eBPF is now more than just an observability tool.
    - Rather, eBPF provides a safe, efficient, and flexible way for privileged
      users to extend the Linux kernel.
    - In other words, eBPF turns Linux into a programmable kernel.

- In the security space, a good example of this paradigm shift is the Linux 5.7
  KRSI framework. KRSI allows BPF programs to be attached to LSM hooks, which
  can then make security decisions and generate audit logs in real time. bpfbox
  uses these BPF LSM programs to implement its policy enforcement mechanism.

# Slide 8: bpfbox Implementation

- In userspace, bpfbox runs as a privileged daemon that leverages the bcc
  framework for Python to compile and load its BPF programs into the kernel.

- All of bpfbox's kernelspace components are written in eBPF, and utilize several
  classes of BPF programs.
  - Tracepoints provide a stable interface for attaching to various kernelspace
    mechanisms such as system calls and the scheduler.
  - kprobes and uprobes provide an interface for dynamically attaching onto
    kernelspace and userspace function calls respectively.
  - Finally, LSM probes allow BPF programs to be attached to LSM hooks, which can
    then be used to enforce security policy and generate audit logs.

- Thanks to eBPF, bpfbox's implementation is light-weight, flexible,
  and production-safe. It works out of the box on any vanilla Linux kernel
  version 5.8 or higher.

# Slide 9: bpfbox Architecture

- When bpfbox starts, it loads its BPF programs into the kernel and creates
  maps to hold policy and process state. It then parses the user-defined policy
  and loads the corresponding data into the policy maps.

- Whenever a sandboxed application requests access to a security-sensitive
  resource, the access request traps to an LSM probe which uses the currently
  active policy and state of the running process to inform its policy decision.

# Slide 10: Policy Design Goals

- When we were designing our policy language, we had four main design goals in mind.

1. bpfbox policy should be simple enough that it can be used for ad hoc process
   confinement.

2. bpfbox policy should be application transparent, and shouldn't require any
   changes to the source code of the confined application.

3. bpfbox policy should be flexible, offering optional layers of granularity, so
   that it can provide optional fine-grained protection while its base
   functionality remains easy to use.

4. bpfbox policy should follow the principle of least privilege, and it should
   be difficult to write an insecure policy.

# Slide 11: Rules and Directives

- bpfbox policy at its core consists of a series of rules that can be optionally
  augmented by directives.

- Rules specify access to various system objects, and generally take an
  identifier, such as a pathname, to specify the object and one or more modes of
  access.

- Directives are used to augment blocks of rules and are written using a special
  syntax. In general, directives can either be used to specify an action that
  should be taken on a group of rules, or to specify additional context for
  a group of rules.

# Slide 12: Taints and Transitions

- bpfbox doesn't start enforcing its policy on a given process until that
  process enters an insecure state, which we call the "tainted" state. A special
  taint directive is used to specify the conditions under which a process should
  become tainted. A tainted process will remain tainted until it either
  terminates or transitions to another profile.

- Transitioning to another profile can only occur when a process makes an
  allowed execve call that has been explicitly marked with the transition
  directive.

- In this example, we have a simplified policy for a web facing daemon, that
  taints itself upon performing any IPv4 or IPv6 networking. It is allowed
  to execute a helper application, transitioning profiles when it does so.

# Slide 13: Policy at the Function Call Level

- One of bpfbox's more unique features is its ability to augment policy with
  additional context. A special "func" directive is used to specify that a block
  of rules should only apply within a call to a given userspace function.

- The "kfunc" directive does the same thing, but with a kernel function instead.

- In this example, we have a simplified profile for a login program. It is
  allowed to read `/etc/passwd` and `/etc/shadow` within a call to the
  `check_password` function. It is allowed to read and append to `/etc/shadow`
  and `/etc/passwd` within a call to the `add_user` function. In all other
  instances, these access patterns would be denied.

# Slide 14: Performance Evaluation (Methodology)

- We evaluated bpfbox's performance impact on the running system using a variety
  of benchmarking tests. In each test, we compared bpfbox's overhead to the
  overhead of AppArmor.

- The Phoronix OSBench suite measures basic operating system functionality,
  such as spawning processes, memory allocations, and filesystem accesses. These
  tests generally strongly reflect the kind of events that bpfbox interposes on,
  and thus can provide a general idea of bpfbox's overhead on the overall
  system.

- The Phoronix Apache suite measures the performance of the Apache httpd
  webserver based on how many packets it can process per second. This gives us
  an idea of bpfbox's relative overhead on a busy webserver workload.

- Finally, we used an ad-hoc kernel compilation benchmark to measure the
  userspace, kernelspace, and elapsed-time overhead of Linux kernel compilation.
  This test involves a heavy workload that spawns a large amount of processes,
  which is useful for measuring bpfbox's performance overhead on a very
  computationally heavy task.


# Slide 15: Performance Evaluation (Results)

- Our OSBench results showed that, in the average case, bpfbox's overhead is
  roughly equivalent to AppArmor, while in the worst case bpfbox performs
  significantly better than AppArmor.

- Our Apache httpd benchmarks showed that bpfbox and AppArmor exhibited roughly
  equivalent overhead under the artificial workload,

- Our kernel compilation benchmarks showed that, in the average case, bpfbox is
  roughly equivalent to AppArmor, while in the worst case bpfbox performs better
  in kernelspace overhead and worse in userspace overhead.

- In all cases, bpfbox exhibited less than 15% overhead on the running system,
  which we find to be acceptable in practice.

# Slide 16: Contributions

- In summary, we have implemented the first policy enforcement engine written in
  eBPF.

- Our solution allows for the integration of userspace and kernelspace system
  state with LSM layer enforcement, which enables the creation of very
  fine-grained policy, transparent to the target application.

- Finally, we were able to prototype a simple policy language that can be
  used for ad hoc process confinement, but which can optionally be used to
  augment rules with system state for fine-grained protection.

# Slide 17: Acknowledgements

- We would like to thank Alexei Starovoitov and Daniel Borkmann for spearheading
  the creation of the new BPF, K.P. Singh for writing the KRSI patch that
  enabled BPF LSM programs, my fellow bcc contributors for creating an awesome
  eBPF framework, and the anonymous CCSW reviewers for providing valuable
  feedback on our paper.

- bpfbox is free and open source software, and is available at the link below.
