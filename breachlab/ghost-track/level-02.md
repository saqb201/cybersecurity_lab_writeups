# BreachLab Ghost Track — Level 2: In The Shadows

**Platform:** breachlab.org  
**Difficulty:** Easy  
**Category:** Linux / File System Enumeration  
**Date:** August 10, 2026  

---

## Challenge Description

> KAEL compartmentalized everything. The investigation folder has reports — but not what you need.
> He kept active leads somewhere else. Somewhere less obvious.
>
> **Goal:** Retrieve the password for ghost3  
> **Connect:** `ssh ghost3@204.168.229.209 -p 2222`
>
> Hints provided:
> - https://man7.org/linux/man-pages/man1/ls.1.html
> - https://man7.org/linux/man-pages/man1/find.1.html

---

## Initial Enumeration

Logged in as ghost2 and ran a basic `ls`:

```bash
ghost2@breachlab:~$ ls
investigation
```

Only one visible folder. Moved into it:

```bash
ghost2@breachlab:~$ cd investigation/
ghost2@breachlab:~/investigation$ ls
report.txt  summary.txt
```

Read both files:

```bash
ghost2@breachlab:~/investigation$ cat report.txt
INCIDENT REPORT — Q1 2026
Status: Classified
Prepared by: KAEL
Summary: Unauthorized access detected on segment C.
Response: Ongoing. Active leads compartmentalized.
Full details filed separately.

ghost2@breachlab:~/investigation$ cat summary.txt
OPERATIONAL SUMMARY
===================
Operation: GHOST WATCH
Status: Active
All active source files have been compartmentalized
and moved to a separate location.
This document contains no credentials.
```

Both files explicitly state credentials are stored elsewhere.

---

## Observations

The word "compartmentalized" in both files and the challenge description is a deliberate hint pointing toward **hidden files and directories**.

In Linux, any file or directory whose name starts with a `.` (dot) is hidden from a standard `ls` command. You need `ls -la` to reveal them.

Ran `ls -la` in the home directory:

```bash
ghost2@breachlab:~$ ls -la
total 72
drwx------ 1 ghost2 ghost2 4096 Aug  9 13:41 .
drwxr-xr-x 1 root   root   4096 Jun 22 13:41 ..
-rw-r--r-- 1 ghost2 ghost2  220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 ghost2 ghost2 3771 Jan  6  2022 .bashrc
drwx------ 2 ghost2 ghost2 4096 Aug  9 13:41 .cache
drwxrwxr-x 3 ghost2 ghost2 4096 Jul 11 23:04 .local
-rw-r----- 1 ghost2 ghost2  153 Jun 22 13:41 .memo
-rw-r--r-- 1 ghost2 ghost2  807 Jan  6  2022 .profile
drwx------ 2 ghost2 ghost2 4096 Jun 24 02:11 .ssh
drwxrwxr-x 3 ghost2 ghost2 4096 Jul  8 17:26 .terminfo
drwxr-x--- 1 ghost2 ghost2 4096 Jul  6 08:54 investigation
```

Several hidden items appeared. Also noticed `.memo` with restricted permissions — read it:

```bash
ghost2@breachlab:~$ cat .memo
NOTE TO SELF — KAEL
The work happens off the main paths.
Compartmentalization is the only real opsec.
If it's in plain sight, it's not worth finding.
```

Another deliberate hint — the answer is off the main paths.

---

## Rabbit Holes

Investigated every hidden directory in the home folder:

```bash
# .cache — only contained an empty motd file, nothing useful
ghost2@breachlab:~$ cd .cache && ls -la
-rw-r--r-- 1 ghost2 ghost2 0 Aug  9 13:41 motd.legal-displayed

# .local — only contained kitty terminal integration files
ghost2@breachlab:~/.local/share/kitty-ssh-kitten/shell-integration$ ls
fish  zsh

# .ssh — only contained known_hosts, no credentials
ghost2@breachlab:~/.ssh$ ls
known_hosts

# .terminfo — only terminal config files
ghost2@breachlab:~/.terminfo$ ls
78
```

None of these contained the password. The key realization: I had only used plain `ls` inside the `investigation` folder — never `ls -la`. Hidden directories could exist there too.

---

## Solution

Went back into `investigation/` and ran `ls -la` this time:

```bash
ghost2@breachlab:~/investigation$ ls -la
total 40
drwxr-x--- 1 ghost2 ghost2 4096 Jul  6 08:54 .
drwx------ 1 ghost2 ghost2 4096 Aug  9 13:41 ..
drwxr-x--- 1 ghost2 ghost2 4096 Jul  2 16:33 .leads
-rw-r----- 1 ghost2 ghost2  201 Jun 22 13:41 report.txt
-rw-r----- 1 ghost2 ghost2  205 Jun 22 13:41 summary.txt
```

A hidden directory `.leads` appeared that plain `ls` had completely concealed.

```bash
ghost2@breachlab:~/investigation$ cd .leads
ghost2@breachlab:~/investigation/.leads$ ls -la
total 40
drwxr-x--- 1 ghost2 ghost2 4096 Jul  2 16:33 .
drwxr-x--- 1 ghost2 ghost2 4096 Jul  6 08:54 ..
-rw-r----- 1 ghost2 ghost2  13 Jun 22 13:41 .source_alpha
-rw-r----- 1 ghost2 ghost2  13 Jun 22 13:41 .source_beta
-rw-r----- 1 ghost2 ghost2  15 Jun 22 13:41 .source_omega
```

Three hidden files inside a hidden directory — double concealment. Read all three:

```bash
ghost2@breachlab:~/investigation/.leads$ cat .source_alpha
[REDACTED]

ghost2@breachlab:~/investigation/.leads$ cat .source_beta
[REDACTED]

ghost2@breachlab:~/investigation/.leads$ cat .source_omega
[REDACTED — solve it yourself at breachlab.org]
```

`.source_alpha` and `.source_beta` are decoys. `.source_omega` contains the real password.

```bash
┌──(kali㉿kali)-[~]
└─$ ssh ghost3@204.168.229.209 -p 2222
(ghost3@204.168.229.209) Password: [REDACTED]

# Successfully logged in as ghost3 ✓
```

---

## Why It Worked

Linux hides any file or directory whose name begins with a dot (`.`). Standard `ls` skips them entirely.

| Command | What it shows |
|---------|--------------|
| `ls` | Visible files only |
| `ls -a` | All files including hidden |
| `ls -la` | All files + permissions + size + date |

The password was buried two levels deep — inside a hidden subdirectory `.leads` nested inside a visible directory `investigation`. Using plain `ls` at any point in that path would have made it invisible.

The decoy files `.source_alpha` and `.source_beta` are a real-world technique — misdirection to slow down investigators and test whether you read carefully.

**The `find` command is an even faster approach for this type of challenge:**

```bash
find ~ -name ".*" -type f 2>/dev/null
```

This recursively finds ALL hidden files from your home directory in one command — much faster than manually exploring directories one by one.

---

## Key Takeaway

Always use `ls -la` instead of plain `ls` when enumerating Linux systems — hidden dotfiles and directories are one of the most common places credentials, configs, and sensitive data are stored in both CTFs and real-world penetration tests.

---

*Part of my BreachLab Ghost Track series — documenting every level as I complete it.*  
*GitHub: https://github.com/saqb201/cybersecurity_lab_writeups*

*Password redacted in compliance with breachlab.org rules.*
