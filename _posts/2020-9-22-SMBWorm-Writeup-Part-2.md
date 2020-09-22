---
layout: post
title: SMB Worm Writeup (Part 2)
---

Welcome to the part 2 of reversing an SMB Worm. In [part 1](https://evilsapphire.github.io/SMBWorm-Writeup-Part-1/) we observed the worm generating random Public IPs, then checking whether data can be sent to over to the Public via calls to `connect` and 'select` WINAPIs, and if the check succeded it generated the SMB Share format of the IP by appending '\\\\' before the Public IP string and passed this `\\\\\<Public-IP>' to `sub_4012B0`. In this post we are going to take a look at how exactly the worm is propagating itself over SMB connections.
