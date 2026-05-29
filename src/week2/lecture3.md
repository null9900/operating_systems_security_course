# AppArmor (Mandatory Access Control)

|||objectives
After this lecture, you should be able to answer the following:
- What is MAC and how is it different from DAC?
- What is AppArmor and how does it work?
- How to create, manage and debug AppArmor profiles?
|||

### DAC vs MAC

DAC asks: **is this user allowed to access this file?**

MAC (Mandatory Access Control) asks a different question: **is this program allowed to do this action?**

### What is AppArmor?
AppArmor (Application Armor) is a Linux Security Module (LSM) that implements MAC. It is the default MAC system on Ubuntu, Debian and SUSE. The alternative is SELinux, which is the default on RHEL, Fedora and CentOS. They solve the same problem differently.

```bash
# This shows you which profiles are loaded
sudo aa-status
```

### Walkthrough: confining a backup script

```bash
#!/bin/bash
mkdir -p ./backups ./myproject
echo "some data" > ./myproject/index.html
echo "$(date): backup started" >> ./backup.log
tar cf ./backups/www-backup.tar ./myproject/
echo "$(date): backup finished" >> ./backup.log
```

```bash
chmod +x backup.sh
```

Right now this script is unconfined. It can do anything root can do. It could read `/etc/shadow`, delete files in `/home/`, open network connections. Nothing stops it.

In terminal 1:
```bash
sudo aa-genprof ./backup.sh
```

In terminal 2, run the script:
```bash
./backup.sh
```

Allow everything that the script legitimately needs. When done, press **F** to finish. The profile is now in enforce mode.


**The generated profile**

```bash
sudo cat /etc/apparmor.d/home.null.apparmor.backup.sh
```

```
include <tunables/global>

/home/null/apparmor/backup.sh {
  include <abstractions/base>
  include <abstractions/bash>
  include <abstractions/consoles>
  include <abstractions/opencl-pocl>

  /etc/group r,
  /etc/ld.so.cache r,
  /etc/locale.alias r,
  /etc/nsswitch.conf r,
  /etc/passwd r,
  /home/null/apparmor/backup.sh r,
  /proc/filesystems r,
  /usr/bin/bash ix,
  /usr/bin/date mrix,
  /usr/bin/mkdir mrix,
  /usr/bin/tar mrix,
  owner /home/*/apparmor/backup.log w,
  owner /home/*/apparmor/backups/ r,
  owner /home/*/apparmor/backups/www-backup.tar rw,
  owner /home/*/apparmor/myproject/ r,
  owner /home/*/apparmor/myproject/index.html r,
  owner /home/*/apparmor/myproject/index.html w,
}
```

This is the profile. Everything the script is allowed to do is listed here. Everything not listed is denied by default. AppArmor is a **whitelisting** system.

### Profile syntax

**Abstractions**

```
include <abstractions/base>
include <abstractions/bash>
include <abstractions/consoles>
```

Abstractions are reusable sets of rules. For example, `base` covers shared libraries and basic system files.

The `/etc/group`, `/etc/ld.so.cache`, `/etc/passwd` rules are also there because the system needs these files to resolve usernames, load libraries, etc.


**File rules**

Each file rule is a path followed by permission flags:

```
owner /home/*/apparmor/backup.log r,
owner /home/*/apparmor/backup.log w,
```

| Flag | Meaning |
|------|---------|
| r | Read the file |
| w | Write to the file |
| a | Append to the file |
| x | Execute the file |
| m | Memory map the file as executable |

The `owner` keyword means this rule only applies when the user running the program actually owns the file. Without `owner`, the rule would apply regardless of who owns it.

**Execution rules**

```
/usr/bin/bash ix,
/usr/bin/date mrix,
/usr/bin/mkdir mrix,
/usr/bin/tar mrix,
```

The `ix` means inherit. When the script runs `tar`, it runs under the same profile as the script.

| Flag | Meaning |
|------|---------|
| ix | **Inherit** - the child runs under the parent's profile |
| px | **Profile** - the child runs under its own separate profile |
| ux | **Unconfined** - the child runs with no AppArmor restrictions |

**Deny rules**

You can add explicit deny rules to a profile:

```
deny /etc/shadow    r,
deny /home/**       rw,
deny network
```

### DAC + MAC together

DAC prevents unauthorized users from running programs. MAC prevents authorized programs from doing unauthorized things.

| | DAC | MAC |
|---|---|---|
| What is controlled? | Access to files based on user identity | What each program is allowed to do |
| Can the user override it? | Yes (owners can `chmod` their files) | No (even root is restricted by policy) |
| Granularity | User/group/others | Per-program rules for files, network, capabilities |

MAC does not replace DAC, but it adds a layer on top. Both checks must pass. If DAC says no, it doesn't matter what MAC says. If DAC says yes, MAC can still say no.

### Useful commands

| Command | Description |
|---------|-------------|
| `aa-status` | Show loaded profiles and their modes |
| `aa-enforce` | Set a profile to enforce mode |
| `aa-complain` | Set a profile to complain mode |
| `aa-disable` | Disable a profile |
| `aa-genprof` | Generate a new profile interactively |
| `apparmor_parser -r` | Reload a profile after editing |

### Exercise

**Fix the profile**

A junior sysadmin wrote this profile for a backup script. It has several security problems. Find and fix them:

```
/usr/local/bin/backup.sh {
    #include <abstractions/base>

    /usr/local/bin/backup.sh    r,

    /var/www/**                 r,
    /backups/**                 rw,

    /usr/bin/tar                ux,
    /usr/bin/gzip               ux,
    /usr/bin/scp                ux,

    /** r,

    network inet stream
}
```

|||quiz
- What is the difference between DAC and MAC?
- What is an AppArmor profile and where are profiles stored?
- Read this rule: `/var/log/myapp/** rw,` - what does it allow?
- AppArmor is a whitelisting system. What does that mean?
|||

<div style="text-align: center; font-size: 0.8em; color: gray; margin-top: 50px;">Maysara Alhindi -- 2026</div>