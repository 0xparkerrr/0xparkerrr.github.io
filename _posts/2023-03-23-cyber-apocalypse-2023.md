---
layout: post
title: Cyber Apocalypse 2023 Writeups
description: Select writeups of challenges from the Cyber Apocalypse 2023 CTF
summary: Select writeups of challenges from the Cyber Apocalypse 2023 CTF.
tags: ctf hackthebox writeups
minute: 1
---
![](/assets/htb/ca2023/banner.png)

# Writeups
Forensics
- [Roten](/writeups/roten)
- [Relic Map](/writeups/relic-map)

# Overview
![](/assets/htb/ca2023/cert.png)
![](/assets/htb/ca2023/team.png)
![](/assets/htb/ca2023/flags.png)

# Reflection
Being my first ever CTF I'm definitely satisfied with my results. Up until the event I only ever got to watch people participate in CTFs so I never got to use the tools myself. Thankfully, I got to use them and learned A LOT.

I learned:
- How to perform a buffer overflow and fill a return address to point to another function (basic)
- How to use pwndbg and Ghidra to reverse engineer programs
- Reverse/pwn quick check tools like file, checksec, and ltrace
- Common CTF tools like RsaCtfTool, Python pwntools
- Certain methodologies and approaches towards web applications
- GraphQL's introspection

My best category was Forensics, getting 6/10 flags. This is because I've practiced a lot on other platforms when it comes to things like packet and malware analysis.

My favorite category was Reversing. Finding how programs work and manipulating them to get an unexpected result kept me engaged and wanting to learn more.