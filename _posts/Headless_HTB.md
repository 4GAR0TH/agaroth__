---
title: HackTheBox - Headless
date: 2024-08-04 12:35:00 +0800
categories: [hackthebox]
tags: [ctf]     # TAG names should always be lowercase
---

Headless

1 STEP SCAN 

We have a webserver 

2 STEP ENUMERATION

We see, we have two directories for our website
we notice that we can only have the /support 
the website doesn't allow the user to connect to the second /dashboard

3 STEP INTERACT LIKE A USER

TRY XSS 
BLOCKED

4 STEP TRY TO HAVE ACCESS TO DASHBOARD

COOKIE

5 STEP LET'S HAVE MORE CONTACT WITH THE SERVER

COMMAND INJECTION
netcat machin 

FIRST FLAG

6 STEP A PRIVILEGE ESCALATION MUST BE PERFORMED

check with sudo -l
understand the syscheck command
initdb.sh
























