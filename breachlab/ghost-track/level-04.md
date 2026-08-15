# BreachLab Ghost Track — Level 4: Signal in the Noise

**Platform:** breachlab.org  
**Difficulty:** Easy  
**Category:** Linux / grep & Text Filtering  
**Date:** August 10, 2026  

---

## Challenge Description

> KAEL dumped everything into the vault.
> Hundreds of log entries. One of them isn't like the others.
> The real signal has a different format. Find it.
>
> **Goal:** Retrieve the password for ghost5  
> **Connect:** `ssh ghost5@204.168.229.209 -p 2222`
>
> Hints provided:
> - https://man7.org/linux/man-pages/man1/grep.1.html
> - https://ryanstutorials.net/linuxtutorial/piping.php

---

## Initial Enumeration

Logged in as ghost4. Basic `ls` showed one directory:

```bash
ghost4@breachlab:~$ ls
vault
ghost4@breachlab:~$ cd vault/
ghost4@breachlab:~/vault$ ls
record_0001  record_0073  record_0145 ... record_0500
```

500 individual record files. Opening them one by one is not an option — this challenge is explicitly about filtering large datasets efficiently.

---

## Observations

The challenge says:
- "Hundreds of log entries"
- "One of them isn't like the others"
- "The real signal has a **different format**"

This is a direct pointer to `grep` — the tool for searching through text. The hints also confirm this.

The key insight: with 500 files, you need to search **all of them at once** using a wildcard (`*`) and pipe the output through `grep` to filter for specific keywords.

---

## Rabbit Holes

**First attempt — searched for "password" (lowercase):**

```bash
ghost4@breachlab:~/vault$ cat record_0* | grep "password"
[2026-03-28 02:47:13] password=[REDACTED]
[2026-03-28 02:47:13] password=[REDACTED]
[2026-03-28 02:47:13] password=[REDACTED]
[2026-03-28 02:47:13] password=[REDACTED]
[2026-03-28 02:47:13] password=[REDACTED]
```

Five results — all with the same timestamp and same format. These are decoys. The challenge said "one of them isn't like the others" — five identical-format results is suspicious. None of these are the real password.

**Second attempt — tried "Password" (capital P):**

```bash
ghost4@breachlab:~/vault$ cat record_0* | grep "Password"
```

No results.

**Third attempt — tried "Credentials" and "credentials":**

```bash
ghost4@breachlab:~/vault$ cat record_0* | grep "Credentials"
ghost4@breachlab:~/vault$ cat record_0* | grep "credentials"
```

No results either.

---

## Solution

Tried uppercase "CREDENTIAL" — and found the one entry with a completely different format:

```bash
ghost4@breachlab:~/vault$ cat record_0* | grep "CREDENTIAL"
[CLASSIFIED] CREDENTIAL: [REDACTED — solve it yourself at breachlab.org]
```

One result. Different format from everything else — `[CLASSIFIED]` prefix instead of a timestamp, `CREDENTIAL:` instead of `password=`. This is the signal in the noise.

Confirmed by logging into ghost5:

```bash
┌──(kali㉿kali)-[~]
└─$ ssh ghost5@204.168.229.209 -p 2222
(ghost5@204.168.229.209) Password: [REDACTED]

# Successfully logged in as ghost5 ✓
```

---

## Why It Worked

**`grep`** searches for a pattern inside text. Combined with **piping (`|`)** and **wildcards (`*`)**, it becomes a powerful filtering tool across hundreds of files at once.

The command breakdown:

```bash
cat record_0* | grep "CREDENTIAL"
```

| Part | What it does |
|------|-------------|
| `cat record_0*` | Reads ALL files matching `record_0*` and outputs them |
| `\|` | Pipes all that output into the next command |
| `grep "CREDENTIAL"` | Filters and shows only lines containing "CREDENTIAL" |

**Why the decoy "password=" entries didn't work:**  
The challenge deliberately seeded fake results with lowercase `password=` to waste your time if you only search once. The real entry used a completely different keyword (`CREDENTIAL`) and a different format (`[CLASSIFIED]`). This mirrors real-world log analysis — attackers and defenders both know that obvious keywords get filtered first.

**Case sensitivity matters in grep:**  
`grep "password"` and `grep "Password"` and `grep "CREDENTIAL"` are three different searches. By default grep is case-sensitive. You can use `grep -i` to search case-insensitively, but then you lose precision.

**A smarter one-liner for this challenge:**

```bash
grep -r "CREDENTIAL" ~/vault/
```

The `-r` flag makes grep search recursively through a directory — no need to `cat` all files first. Cleaner and faster.

---

## Key Takeaway

`grep` with pipes and wildcards is one of the most essential tools in a SOC analyst's daily workflow — real log files contain thousands of entries and finding the anomaly that "doesn't look like the others" is exactly what threat hunting and incident response require.

---

*Part of my BreachLab Ghost Track series — documenting every level as I complete it.*  
*GitHub: https://github.com/saqb201/cybersecurity_lab_writeups*

*Password redacted in compliance with breachlab.org rules.*
