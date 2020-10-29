# Slide 0: Title Slide

- I'm William Findlay from Carleton University and today I'll be talking about
  bpfbox and how it leverages the new Kernel Runtime Security
  Instrumentation feature in eBPF to implement process confinement.

# Slide 1: bpfbox at a Glance

- At a glance, bpfbox represents a new approach to process confinement under the
  Linux kernel, enabled by eBPF.

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
  opportunities for improving operating system security.

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

# Slide 4: bpfbox Implementation

- In userspace, bpfbox runs as a privileged daemon that leverages the bcc
  framework for Python to compile and load its BPF programs into the kernel.

- All of bpfbox's kernelspace components are written in eBPF, and utilize several
  classes of BPF programs.
  - We use LSM probes for policy enforcement
  - kprobes for tracking kernelspace state
  - uprobes for tracking userspace state

- Thanks to eBPF, bpfbox's implementation is light-weight, flexible,
  and production-safe. It works out of the box on any vanilla Linux kernel
  version 5.8 or higher.

# Slide 5: Rules and Directives

- bpfbox policy at its core consists of a series of rules that can be optionally
  augmented by directives.

- Rules specify access to various system objects, and generally take an
  identifier, such as a pathname, to specify the object and one or more modes of
  access.

- Directives are used to augment blocks of rules and are written using a special
  syntax. In general, directives can either be used to specify an action that
  should be taken on a group of rules, or to specify additional context for
  a group of rules.

# Slide 6: Taints and Transitions

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

# Slide 7: Policy at the Function Call Level

- One of bpfbox's more unique features is its ability to augment policy with
  additional context. A special "func" directive is used to specify that a block
  of rules should only apply within a call to a given userspace function.

- The "kfunc" directive does the same thing, but with a kernel function instead.

- In this example, we have a simplified profile for a login program. It is
  allowed to read `/etc/passwd` and `/etc/shadow` within a call to the
  `check_password` function. It is allowed to read and append to `/etc/shadow`
  and `/etc/passwd` within a call to the `add_user` function. In all other
  instances, these access patterns would be denied.

# Slide 8: Acknowledgements

- We would like to thank Alexei Starovoitov and Daniel Borkmann for spearheading
  the creation of the new BPF, K.P. Singh for writing the KRSI patch that
  enabled BPF LSM programs, my fellow bcc contributors for creating an awesome
  eBPF framework, and the anonymous CCSW reviewers for providing valuable
  feedback on our paper.

- bpfbox is free and open source software, and is available at the link below.
