# Seccomp: Restricting System Calls

|||objectives
After this lecture, you should be able to answer the following:
- What is seccomp and what problem does it solve?
- How does a seccomp filter decide what to allow or block?
- How do you build a filter, and how does `strace` help?
|||

### What is seccomp?
**seccomp** (secure computing mode) is a Linux kernel feature that lets a program restrict which system calls it is allowed to make. 
Seccomp allows developers to restrict what system calls their application is allowed to make, keeping only the few it actually needs.

To install Seccomp:
```
sudo apt install libseccomp-dev
```

### The allowlist model
A seccomp filter works on the same whitelisting principle as the AppArmor profiles we built. You set a default action (usually "kill the process"), and then explicitly allow the system calls the program legitimately needs. Everything you don't list is denied by default.
When a system call happens, the kernel checks it against the filter and applies an action:

| Action | What it does |
|--------|-------------|
| ALLOW | Let the system call run |
| KILL_PROCESS | Kill the whole process |
| KILL | Kill the offending thread |
| ERRNO(x) | Block it and return an error code instead |
| LOG | Allow it, but log that it happened |
| TRAP | Send a signal (SIGSYS) the program can catch |

### Building a filter
```c
#include <seccomp.h>
#include <stdio.h>
#include <unistd.h>
int main(void) {
    // Default action: kill the process on any syscall we didn't allow
    scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_KILL_PROCESS);
    // Allow only what we need
    seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(read), 0);
    seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 0);
    seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit_group), 0);
    // After this, the filter is active and enforced
    seccomp_load(ctx);
    // write is allowed
    write(1, "hello\n", 6);
    // open is not allowed
    open("/etc/passwd", 0);
    write(1, "you will never see this\n", 24);
    return 0;
}
```

Compile and run it:
```bash
gcc -o demo demo.c -lseccomp
./demo
```

```bash
strace ./myprogram
```

```bash
# Watch seccomp log entries while testing (LOG mode)
sudo dmesg -w | grep -i seccomp
```

### Filtering by argument
Seccomp can look at a system call's arguments, not just its number. Because the underlying filter is a BPF program running over the syscall data, you can allow a call only when an argument matches a condition. libseccomp exposes this with argument comparators like `SCMP_A0`, `SCMP_A1` (argument 0, 1, ...).

For example, allow `write` only to file descriptor 1 (stdout), and block writing to anything else:
```c
// allow write only when arg0 == 1  (fd 1 = stdout)
seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 1, SCMP_A0(SCMP_CMP_EQ, 1));
```

### Seccomp, AppArmor, and DAC together

We now have three layers of defense, and they answer different questions:

| Layer | Controls | Question it answers |
|-------|----------|--------------------|
| DAC | File access by user | Can this user touch this file? |
| AppArmor (MAC) | What a program may access | Can this program open this path / use the network? |
| seccomp | Which system calls a program may make | Can this program make this kernel request at all? |

Imagine a compromised web server. DAC limits it to the `www-data` user's files. AppArmor confines it to its own directories and blocks unexpected network access. And seccomp means that even if the attacker runs code inside the process, they can't call `execve` to launch a shell or `socket` to open a backdoor.

|||quiz
- Why is restricting system calls such a powerful way to contain a program?
- What does it mean that seccomp uses an "allowlist" model?
- How would you use `strace` to help build a seccomp allowlist?
- How can a seccomp filter allow a syscall only for a specific argument value? Give an example.
|||

<div style="text-align: center; font-size: 0.8em; color: gray; margin-top: 50px;">Maysara Alhindi -- 2026</div>