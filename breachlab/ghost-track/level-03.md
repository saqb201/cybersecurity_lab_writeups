# BreachLab Ghost Track — Level 3: Access Denied

**Platform:** breachlab.org  
**Difficulty:** Easy  
**Category:** Linux / File Permissions & Groups  
**Date:** August 10, 2026  

---

## Challenge Description

> KAEL structured his storage by access level.
> Not everything is readable by everyone.
> Know who you are. Know what you can reach.
>
> **Goal:** Retrieve the password for ghost4  
> **Connect:** `ssh ghost4@204.168.229.209 -p 2222`
>
> Hints provided:
> - https://man7.org/linux/man-pages/man1/id.1.html
> - https://man7.org/linux/man-pages/man1/find.1.html
> - https://man7.org/linux/man-pages/man1/chmod.1.html

---

## Initial Enumeration

Logged in as ghost3. Basic `ls` showed one file:

```bash
ghost3@breachlab:~$ ls
map.txt
```

Read it immediately — the challenge basically gives you a map:

```bash
ghost3@breachlab:~$ cat map.txt
KAEL'S STORAGE LAYOUT
=====================
Recovered from workstation. Partially redacted.

  /var/intel/public/   — world readable
  /var/intel/ops/      — restricted
  /var/intel/archive/  — root only

Access follows the group scheme. The kernel will
tell you what you are, if you ask it.

— KAEL
```

Three directories. Three access levels. The map tells us exactly where to look and hints to check our own identity with `id` (the kernel will tell you what you are).

---

## Observations

The map gives a clear hierarchy:
- `/var/intel/public/` — anyone can read
- `/var/intel/ops/` — restricted to a specific group
- `/var/intel/archive/` — root only, off limits

The phrase *"The kernel will tell you what you are, if you ask it"* is a direct hint to run the `id` command, which shows your current user and group memberships.

Ran `ls -la` on `/var/intel/` to see the actual permissions:

```bash
ghost3@breachlab:/var/intel$ ls -la
total 24
drwxr-xr-x 1 root root     4096 Jun 22 13:41 .
drwxr-xr-x 1 root root     4096 Jun 22 13:41 ..
drwx------ 2 root root     4096 Jun 22 13:41 archive
drwxr-x--- 1 root analysts 4096 Jun 22 13:41 ops
drwxr-xr-x 1 root root     4096 Jun 22 13:41 public
```

Breaking down the permissions:

| Directory | Permissions | Owner | Group | Meaning |
|-----------|-------------|-------|-------|---------|
| `archive` | `drwx------` | root | root | Root only — no access |
| `ops` | `drwxr-x---` | root | analysts | Group `analysts` can read |
| `public` | `drwxr-xr-x` | root | root | Everyone can read |

The `ops` directory is owned by the group `analysts`. If ghost3 is a member of that group, we can read it.

---

## Rabbit Holes

**Tried `/var/intel/public/` first** — world readable but contained nothing useful:

```bash
ghost3@breachlab:/var/intel/public$ cat report_q1.txt
OPEN INTELLIGENCE REPORT — Q1 2026
Classification: Public
This document contains no operational credentials.
For restricted files, consult your access group.
```

**Tried to access `/var/intel/archive/`** — permission denied as expected:

```bash
ghost3@breachlab:/var/intel$ cd archive/
-bash: cd: archive/: Permission denied
```

Root only. Dead end confirmed.

---

## Solution

Navigated directly to `/var/intel/ops/` and it worked — meaning ghost3 is already a member of the `analysts` group:

```bash
ghost3@breachlab:/var/intel$ cd ops
ghost3@breachlab:/var/intel/ops$ ls
access_codes.dat  operative_list.txt
```

Two files. Read `operative_list.txt` first:

```bash
ghost3@breachlab:/var/intel/ops$ cat operative_list.txt
INDEX OF ACTIVE OPERATIONS
Classification: Analyst Only
See access_codes.dat for current credentials.
```

Points directly to `access_codes.dat`:

```bash
ghost3@breachlab:/var/intel/ops$ cat access_codes.dat
P3rm1ss10ns_M4tt3r
```

Password found: `P3rm1ss10ns_M4tt3r`

Confirmed by logging into ghost4:

```bash
┌──(kali㉿kali)-[~]
└─$ ssh ghost4@204.168.229.209 -p 2222
(ghost4@204.168.229.209) Password: P3rm1ss10ns_M4tt3r

# Successfully logged in as ghost4 ✓
```

---

## Why It Worked

Linux permissions control access at three levels: **owner**, **group**, and **others**. Every file and directory has all three defined.

Reading the permission string `drwxr-x---`:

```
d  rwx  r-x  ---
│   │    │    │
│   │    │    └── Others: no access
│   │    └─────── Group: read + execute (can enter directory)
│   └──────────── Owner: full access
└──────────────── d = directory
```

The `ops` directory had `r-x` for its group (`analysts`) — meaning members of that group can list and enter the directory. Since ghost3 was already assigned to the `analysts` group by the system, access was granted automatically.

**The `id` command** (hinted in the challenge) would have confirmed this upfront:

```bash
ghost3@breachlab:~$ id
uid=1003(ghost3) gid=1003(ghost3) groups=1003(ghost3),1004(analysts)
```

This shows ghost3 belongs to both `ghost3` and `analysts` groups — explaining why `ops` was accessible.

**Permission values quick reference:**

| Symbol | Value | Meaning |
|--------|-------|---------|
| `r` | 4 | Read |
| `w` | 2 | Write |
| `x` | 1 | Execute / Enter directory |
| `-` | 0 | No permission |

So `rwxr-x---` = `750` in numeric notation.

---

## Key Takeaway

Always check Linux permissions with `ls -la` and your own identity with `id` before assuming you can or cannot access something — group membership silently grants access to restricted directories, which is a critical concept in both privilege escalation and real-world SOC investigations.

---

*Part of my BreachLab Ghost Track series — documenting every level as I complete it.*  
*GitHub: https://github.com/saqb201/cybersecurity_lab_writeups*
