# BreachLab Ghost Track — Level 1: Name Game

**Platform:** breachlab.org  
**Difficulty:** Easy  
**Category:** Linux / Shell Quoting  
**Date:** August 10, 2026  

---

## Challenge Description

> KAEL was paranoid. He named his files in ways that make the shell fight you.
> Read the MANIFEST. Then figure out how.
>
> **Goal:** Retrieve the password for ghost2  
> **Connect:** `ssh ghost2@204.168.229.209 -p 2222`
>
> Hints provided:
> - https://man7.org/linux/man-pages/man1/cat.1.html
> - https://tldp.org/LDP/Bash-Beginners-Guide/html/sect_03_03.html
> - https://ss64.com/bash/syntax-quoting.html

---

## Initial Enumeration

After logging in as ghost1, the first thing I did was list the directory contents:

```bash
ghost1@breachlab:~$ ls
 -   --help   MANIFEST  'file name'
```

Something immediately looked wrong. The filenames were:
- `-` (just a dash)
- `--help` (looks like a command flag)
- `MANIFEST` (normal)
- `file name` (has a space in it)

Ran `ls -la` to see full details including hidden files:

```bash
ghost1@breachlab:~$ ls -la
total 88
-rw-r----- 1 ghost1 ghost1   13 Jun 22 13:41  -
-rw-r----- 1 ghost1 ghost1   13 Jun 22 13:41  --help
drwx------ 1 ghost1 ghost1 4096 Aug  9 13:40  .
drwxr-xr-x 1 root   root   4096 Jun 22 13:41  ..
-rw-r----- 1 ghost1 ghost1   13 Jun 22 13:41  ...
-rw-r--r-- 1 ghost1 ghost1  220 Jan  6  2022  .bash_logout
-rw-r--r-- 1 ghost1 ghost1 3771 Jan  6  2022  .bashrc
drwx------ 2 ghost1 ghost1 4096 Aug  9 13:40  .cache
-rw-r--r-- 1 ghost1 ghost1  807 Jan  6  2022  .profile
drwx------ 2 ghost1 ghost1 4096 Jul 11 12:17  .ssh
-rw-r----- 1 ghost1 ghost1  228 Apr 17 09:44  MANIFEST
-rw-r----- 1 ghost1 ghost1   15 Jun 22 13:41 'file name'
```

Even more unusual files appeared: `...` (three dots) and standard hidden dotfiles.

Read the MANIFEST first as instructed:

```bash
ghost1@breachlab:~$ cat MANIFEST
```

---

## Observations

Three things stood out immediately:

**1. The filename `-`**  
Running `cat -` tells bash to read from standard input (keyboard), not a file. The dash is a special symbol in Linux that means stdin. It would hang forever waiting for keyboard input.

**2. The filename `--help`**  
Running `cat --help` would display cat's help menu, not the file contents. The shell interprets anything starting with `--` as a command flag.

**3. The filename `file name`** (with a space)  
Running `cat file name` would make bash think you want to read TWO files — one called `file` and one called `name`. Neither exists, so it would throw an error.

The challenge is testing whether you know how to handle filenames that break normal shell behavior.

---

## Rabbit Holes

```bash
# This hangs forever — reads from keyboard not file
ghost1@breachlab:~$ cat -

# This shows the help menu, not file contents
ghost1@breachlab:~$ cat --help

# This errors — bash sees two separate filenames
ghost1@breachlab:~$ cat file name
cat: file: No such file or directory
cat: name: No such file or directory
```

The hints pointed to bash quoting documentation — that was the key clue.

---

## Solution

The `file name` file (with the space) contains the actual password. The fix is wrapping it in single quotes so bash treats the whole thing as one filename:

```bash
ghost1@breachlab:~$ cat 'file name'
[REDACTED — solve it yourself at breachlab.org]
```

Logged in as ghost2 to confirm:

```bash
┌──(kali㉿kali)-[~]
└─$ ssh ghost2@204.168.229.209 -p 2222
(ghost2@204.168.229.209) Password: [REDACTED]

# Successfully logged in as ghost2 ✓
```

---

## Why It Worked

In Linux bash, the shell processes your command **before** passing it to the program. So when you type:

```bash
cat file name
```

Bash splits on the space and sees: `cat` + `file` + `name` — three separate words.

But when you use single quotes:

```bash
cat 'file name'
```

Single quotes tell bash: treat everything inside me as one literal string — no splitting, no interpretation. The shell passes `file name` (with space) as a single filename to cat.

**Three ways to handle problem filenames in Linux:**

| Method | Example | When to use |
|--------|---------|-------------|
| Single quotes | `cat 'file name'` | Filename has spaces or special chars |
| Double quotes | `cat "file name"` | Same, but allows variable expansion inside |
| Backslash escape | `cat file\ name` | Escaping individual characters |
| Relative path with `./` | `cat ./-` | Handles filenames starting with `-` |

**Bonus:** For the `-` filename specifically, the trick is:

```bash
cat ./-
```

Adding `./` makes bash treat it as a path (current directory + dash) instead of a flag.

---

## Key Takeaway

Linux shell quoting is not just syntax — it is a core security and enumeration skill. Attackers name files with special characters to confuse automated tools and analysts. Knowing how to handle `-`, `--flags`, spaces, and special characters in filenames is essential for both CTF work and real-world incident response.

---

*Part of my BreachLab Ghost Track series — documenting every level as I complete it.*  
*GitHub: https://github.com/saqb201/cybersecurity_lab_writeups*

*Password redacted in compliance with breachlab.org rules.*
