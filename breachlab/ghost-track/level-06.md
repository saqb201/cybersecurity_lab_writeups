# BreachLab Ghost Track — Level 6: Ghost in the Machine

**Platform:** breachlab.org  
**Difficulty:** Medium  
**Category:** Linux / Environment Variables & Base64  
**Date:** August 15, 2026  

---

## Challenge Description

> After the breach, KAEL stopped writing secrets to disk.
> He told himself the shell would forget them.
> It doesn't.
>
> **Goal:** Retrieve the password for ghost7  
> **Connect:** `ssh ghost7@204.168.229.209 -p 2222`
>
> Hints provided:
> - https://man7.org/linux/man-pages/man1/env.1.html
> - https://man7.org/linux/man-pages/man1/base64.1.html
> - https://12factor.net/config

---

## Initial Enumeration

Logged in as ghost6. No visible files in home directory:

```bash
ghost6@breachlab:~$ ls -la
total 60
drwx------ 1 ghost6 ghost6 4096 Aug 13 15:31 .
drwxr-xr-x 1 root   root   4096 Jun 22 13:41 ..
-rw-r--r-- 1 ghost6 ghost6  220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 ghost6 ghost6 4419 Jun 22 13:41 .bashrc
drwx------ 2 ghost6 ghost6 4096 Aug  9 13:32 .cache
drwxrwxr-x 3 ghost6 ghost6 4096 Jul  5 12:46 .local
-rw-r--r-- 1 ghost6 ghost6 1455 Jun 22 13:41 .profile
drwx------ 2 ghost6 ghost6 4096 Jul 15 13:07 .ssh
drwxrwxr-x 3 ghost6 ghost6 4096 Aug 14 19:26 .terminfo
```

No README, no vault, no map. Nothing visible. The challenge description says "the shell would forget them" — pointing directly at environment variables.

---

## Observations

The hints point to:
- `env` — prints all current environment variables
- `base64` — decodes base64-encoded strings
- 12factor.net/config — a well-known standard that recommends storing config/secrets in environment variables

The key insight: **environment variables live in memory, not on disk.** Developers and sysadmins often store secrets (API keys, passwords, tokens) in environment variables thinking they are safer than files. They are not — anyone with shell access can read them with `env`.

---

## Rabbit Holes

**Explored `.terminfo` directory first** — found a broken symlink:

```bash
ghost6@breachlab:~$ cd ~/.terminfo/78
ghost6@breachlab:~/.terminfo/78$ ls -la
lrwxrwxrwx 1 ghost6 ghost6 16 Jul  5 12:46 xterm-kitty -> ../x/xterm-kitty
```

The symlink pointed to `../x/xterm-kitty` but that path didn't exist. Tried creating it manually — this was a dead end, not the intended path.

**Tried searching the filesystem:**

```bash
ghost6@breachlab:~$ sudo grep -r "ghost7" / 2>/dev/null | grep -v "proc" | grep -v "sys"
```

No results. The secret is not on disk at all — KAEL meant it literally.

**Checked `/opt/` for any running services:**

```bash
ghost6@breachlab:~$ ls -la /opt/
drwxr-xr-x 1 root root 4096 Jul 18 22:40 ghost-cleanup
drwxr-xr-x 2 root root 4096 Jun 22 13:41 ghost-cron
```

Nothing useful accessible.

---

## Solution

Ran `env` to dump all environment variables for the current shell session:

```bash
ghost6@breachlab:~$ env
SHELL=/bin/bash
METRICS_ENABLED=false
REGION=eu-central-1
API_DIGEST=M252X0wzNGtzXzN2M3J5dGgxbmc=
TRACE_SALT=bW9uaXRvcmluZ19rZXlfZGVsdGE3
RUNTIME_TOKEN=c3lzdGVtX3Rva2VuX2dhbW1hX3Yz
CACHE_SEED=bm90X2FfcmVhbF9jcmVkZW50aWFs
...
```

Several variables contained Base64-encoded strings — identifiable by the `=` padding at the end and the character set used. The environment was full of decoy variables alongside the real one.

Filtered specifically for `API_DIGEST`:

```bash
ghost6@breachlab:~$ env | grep API_DIGEST
API_DIGEST=M252X0wzNGtzXzN2M3J5dGgxbmc=
```

Decoded it with base64:

```bash
ghost6@breachlab:~$ echo "M252X0wzNGtzXzN2M3J5dGgxbmc=" | base64 -d
[REDACTED — solve it yourself at breachlab.org]
```

Password confirmed. Successfully logged into ghost7.

---

## Why It Worked

**Environment variables** are key-value pairs stored in a process's memory. Every running process inherits them from its parent. When you SSH into a machine, your shell session has its own set of environment variables set by the system, the user's `.bashrc`, `.profile`, and any service configuration.

```bash
env          # Print all environment variables
printenv     # Same thing
echo $VARNAME  # Print one specific variable
```

**Base64** is an encoding scheme — not encryption. It converts binary data to ASCII text using 64 characters (A-Z, a-z, 0-9, +, /). It is trivially reversible:

```bash
echo "encoded_string=" | base64 -d   # Decode
echo "plain text" | base64           # Encode
```

Base64-encoded strings are identifiable by:
- Characters only from the set: `A-Z a-z 0-9 + / =`
- Often ending with `=` or `==` padding
- Length always a multiple of 4

**Why this is a real-world attack vector:**

Environment variables containing secrets are extremely common in real systems:

| Common variable names | What they store |
|----------------------|----------------|
| `DATABASE_URL` | Database credentials |
| `API_KEY` | Third-party API secrets |
| `SECRET_KEY` | Application signing keys |
| `AWS_SECRET_ACCESS_KEY` | Cloud provider credentials |

Any attacker who gains shell access immediately runs `env` — it is one of the first commands in post-exploitation enumeration. This is why secrets in environment variables are dangerous when combined with any form of shell access.

**The decoy variables** (`TRACE_SALT`, `RUNTIME_TOKEN`, `CACHE_SEED`) were also Base64-encoded to slow you down and test whether you decoded each one methodically or guessed randomly.

---

## Key Takeaway

Environment variables are not a safe place to store secrets — `env` dumps all of them instantly to anyone with shell access. In real penetration tests and post-exploitation scenarios, `env` is one of the first commands run after gaining a foothold, often revealing API keys, database passwords, and authentication tokens stored by developers following the 12-factor app pattern.

---

*Part of my BreachLab Ghost Track series — documenting every level as I complete it.*  
*GitHub: https://github.com/saqb201/cybersecurity_lab_writeups*
