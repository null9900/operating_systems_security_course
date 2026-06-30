# eBPF Implementation

|||objectives
After this lecture, you should be able to answer the following:
- How do you hook the execve and openat tracepoints to monitor behavior?
- How do you stream events from the kernel to user space with a ring buffer?
- How do you watch for specific programs and specific files instead of everything?
|||

### What we're building

1. Reports every program launched (execve)
2. Reports every file opened (openat)
3. Lets you watch for specific program names and specific files


### Simplist version

```py
#!/usr/bin/python3

from bcc import BPF
import os

WATCH_PROCS = ["nc", "ssh", "bash"]
WATCH_FILES = ["/etc/shadow", "/etc/passwd"]

prog = r"""
BPF_RINGBUF_OUTPUT(events, 1 << 4);   // channel to user space

#define EXEC 1
#define OPEN 2

struct event_t {
    u32 pid;
    u32 type;
    char comm[16];        // the process that triggered the event
    char filename[128];   // the program run, or file opened
};

// Shared by both hooks: fill in an event and send it up.
static __always_inline
void emit(u32 type, const char *fname) {
    struct event_t *e = events.ringbuf_reserve(sizeof(*e));
    if (!e) return;       // buffer full, drop this event
    e->pid  = bpf_get_current_pid_tgid() >> 32;
    e->type = type;
    bpf_get_current_comm(e->comm, sizeof(e->comm));
    bpf_probe_read_user_str(e->filename, sizeof(e->filename), fname);
    events.ringbuf_submit(e, 0);
}

TRACEPOINT_PROBE(syscalls, sys_enter_execve) { 
    emit(EXEC, args->filename); return 0; 
}
TRACEPOINT_PROBE(syscalls, sys_enter_openat) { 
    emit(OPEN, args->filename); return 0; 
}
"""

b = BPF(text=prog)

def proc_match(path):
    if not WATCH_PROCS:
        return True                       
    return os.path.basename(path) in WATCH_PROCS

def file_match(path):
    if not WATCH_FILES:
        return True
    return any(w in path for w in WATCH_FILES)

def handle(ctx, data, size):
    e = b["events"].event(data)
    path = e.filename.decode()
    comm = e.comm.decode()
    if e.type == 1 and proc_match(path):
        print("EXEC  %-7d %-16s %s" % (e.pid, comm, path))
    elif e.type == 2 and file_match(path):
        print("OPEN  %-7d %-16s %s" % (e.pid, comm, path))


b["events"].open_ring_buffer(handle)

print("Watching procs:", WATCH_PROCS or "ALL")
print("Watching files:", WATCH_FILES or "ALL")
print("%-5s %-7s %-16s %s" % ("TYPE", "PID", "COMM", "FILENAME"))
print("Press Ctrl-C to stop.\n")

while True:
    try:
        b.ring_buffer_poll(timeout=100)
    except KeyboardInterrupt:
        print("\nStopped.")
        break
```

|||info
Notice the design choice: The kernel reports every event, and Python decides what to keep. This is the simplest approach and easy to edit. For a high-traffic system you'd instead filter inside the kernel - passing the watchlist in through a hash map and dropping non-matches before they're ever sent - so only the events you care about cross into user space.
|||


### Limitations of eBPF over other solutions

* It often observes but can't cleanly act
* Programs don't survive a reboot
* It doesn't replace kernel modules or drivers
* The verifier limits what you can write

|||quiz
- Which two tracepoints did we hook, and why tracepoints rather than kprobes?
- What is the ring buffer's job in this tool?
- Why can't we deny system calls in this setup?
- How can we expand this to block certain IPs?
|||

<div style="text-align: center; font-size: 0.8em; color: gray; margin-top: 50px;">Maysara Alhindi -- 2026</div>