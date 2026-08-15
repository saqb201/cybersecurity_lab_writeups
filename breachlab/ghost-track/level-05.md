# BreachLab Ghost Track — Level 5: The Listener

**Platform:** breachlab.org  
**Difficulty:** Medium  
**Category:** Linux / Network Services & Port Discovery  
**Date:** August 10, 2026  

---

## Challenge Description

> KAEL stopped trusting files after the incident.
> He ran a service instead. Said if it's not on disk,
> it can't be found. He forgot about the ports.
>
> **Goal:** Retrieve the password for ghost6  
> **Connect:** `ssh ghost6@204.168.229.209 -p 2222`
>
> Hints provided:
> - https://manpages.debian.org/bookworm/netcat-openbsd/nc.1.en.html
> - https://man7.org/linux/man-pages/man1/nmap.1.html

---

## Initial Enumeration

Logged in as ghost5. Only one file visible:

```bash
ghost5@breachlab:~$ ls
README
```

Read it carefully — KAEL's README is the entire brief for this level:

```bash
ghost5@breachlab:~$ cat README
ANALYST WORKSTATION — KAEL
Last active: 2026-03-28 02:47 UTC
Status: Abandoned

I left a service running on this box. Used to be my back channel —
two ports, one tells you how to talk, the other answers if you say
the right word.

I locked down `ss` and `netstat` weeks ago. Don't bother. The kernel
won't help you here. You'll have to knock on doors yourself.

`nc` and `curl` are still on the box. That's all you need.

Find the listener. Read what it tells you. Then answer it correctly.

— KAEL
```

Key information extracted from README:
- There are **two ports** — one informational, one that gives the credential
- `ss` and `netstat` are blocked — standard port discovery tools won't work
- Only `nc` (netcat) and `curl` are available
- You need to "say the right word" to get the credential

---

## Observations

Without `ss` or `netstat`, the options for port discovery are:

1. **`nmap`** — network scanner (not mentioned as blocked)
2. **`/proc/net/tcp`** — kernel file that lists open ports in hex
3. **`nc`** — connect to ports manually and probe them

Tried reading `/proc/net/tcp` first since KAEL said "the kernel won't help you" — this was a hint that `/proc` was still readable but would be hard to interpret (hex ports):

```bash
ghost5@breachlab:~$ cat /proc/net/tcp | grep -v "sl"
   0: 00000000:C03D 00000000:0000 0A ...
   1: 00000000:0016 00000000:0000 0A ...
   2: 00000000:A179 00000000:0000 0A ...
   ...
```

The ports are in hexadecimal. `C03D` = 49213, `0016` = 22, `A179` = 41337 and so on. Readable but slow to decode manually.

---

## Bonus Discovery — Port 41337

Before solving the main challenge, ran nmap on the external IP and noticed an unusual port:

```bash
ghost5@breachlab:~$ nmap 204.168.229.209
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
443/tcp  open  https
2222/tcp open  EtherNetIP-1
2251/tcp open  dif-port
```

Then scanned localhost specifically to find all internal services:

```bash
ghost5@breachlab:~$ nmap localhost -p- | grep open
22/tcp    open  ssh
30003/tcp open  amicon-fpsu-ra
30100/tcp open  rwp
30101/tcp open  unknown
31339/tcp open  unknown
41310/tcp open  unknown
41311/tcp open  unknown
41337/tcp open  unknown
49213/tcp open  unknown
```

Connected to port 41337 out of curiosity:

```bash
ghost5@breachlab:~$ nc localhost 41337

  [ CLASSIFIED — GHOST TRACK BONUS ]

  You found the signal.
  The official brief listed 22 levels.
  You kept looking past the brief.

  OPERATIVE KAEL — STATUS: ACTIVE
  Last known location: PHANTOM network.
  Final message before going dark:

  "The machines you trust every day
   are not what they appear to be.
   Docker. Kubernetes. GitHub Actions.
   The real breach starts in the pipeline."

  NEXT TRACK: PHANTOM
  Status: LIVE — 30 levels of Linux privesc.
```

**Hidden bonus level found.** Port 41337 was not part of the main challenge — it was an easter egg for people who scan beyond what's required. This is exactly the kind of thorough enumeration mindset that separates good security analysts from average ones.

---

## Solution

### Method 1 — Direct port discovery with nmap + nc

Scanned localhost to find all listening services:

```bash
ghost5@breachlab:~$ nmap localhost -p- | grep open
30100/tcp open  rwp
30101/tcp open  unknown
```

Connected to port 30100 first — it identified itself as the informational channel:

```bash
ghost5@breachlab:~$ nc localhost 30100

  GHOST PROTOCOL — CHANNEL A
  This channel is informational only.
  Authentication token: GHOST
  Secure channel: port 30101
  Send the token to receive your credential.
```

Port 30100 gives two critical pieces of information:
- The **authentication token** is `GHOST`
- The **secure channel** is port 30101

Connected to port 30101 and sent the token:

```bash
ghost5@breachlab:~$ nc localhost 30101
AUTHENTICATE: GHOST

  Credential: [REDACTED — solve it yourself at breachlab.org]
```

### Method 2 — /proc/net/tcp manual decode

Read the kernel's raw TCP table and decoded hex ports manually:

```bash
ghost5@breachlab:~$ cat /proc/net/tcp | grep -v "sl"
```

The second column shows local address in format `IP:PORT` in hex. Converting:
- `7594` hex = `30100` decimal → Channel A (informational)
- `7595` hex = `30101` decimal → Channel B (credential)

Both methods lead to the same ports. `/proc/net/tcp` is useful when nmap is not available.

Confirmed by logging into ghost6:

```bash
┌──(kali㉿kali)-[~]
└─$ ssh ghost6@204.168.229.209 -p 2222
(ghost6@204.168.229.209) Password: [REDACTED]

# Successfully logged in as ghost6 ✓
```

---

## Why It Worked

**Netcat (`nc`)** is a raw TCP/UDP connection tool — it connects to any port and sends/receives raw data. It's called the "Swiss Army knife" of networking and is present on almost every Linux system.

```bash
nc localhost 30100    # Connect to port 30100 on localhost
```

Once connected, whatever the service sends is printed to your terminal. Whatever you type is sent to the service.

**The two-port pattern** (one informational + one authenticated) is a real design pattern used in:
- Command and control (C2) server infrastructure
- Custom backdoors and RATs
- Legitimate services with separate info and auth endpoints

**`/proc/net/tcp` hex conversion:**  
Ports in `/proc/net/tcp` are in little-endian hex. Convert to decimal:

| Hex | Decimal |
|-----|---------|
| `7594` | 30100 |
| `7595` | 30101 |
| `A179` | 41337 |
| `C03D` | 49213 |

**Why localhost scan found more ports than external scan:**  
Some services only bind to `127.0.0.1` (localhost) and are not exposed externally. Always scan both the external IP and localhost when you have shell access to a machine.

---

## Key Takeaway

When standard enumeration tools (`ss`, `netstat`) are unavailable, fall back to `nmap`, `/proc/net/tcp`, or manual `nc` probing — and always scan localhost separately from the external IP, as internal services are often hidden from outside but fully accessible from within.

**Bonus lesson:** Thorough enumeration beyond the obvious scope (scanning all ports, not just common ones) revealed a hidden bonus level — the same curiosity that finds easter eggs in CTFs finds hidden attack surfaces in real penetration tests.

---

*Part of my BreachLab Ghost Track series — documenting every level as I complete it.*  
*GitHub: https://github.com/saqb201/cybersecurity_lab_writeups*

*Password redacted in compliance with breachlab.org rules.*
