---
layout: post
title: SMB Worm Writeup
---

This is going to be a write-up on reversing a Malware listed as a Generic Trojan in FabriMagic72's Github Malware sample repo found [here](https://github.com/fabrimagic72/malware-samples). The specific sample is the first one found in the 'Generic Trojan' [folder](https://github.com/fabrimagic72/malware-samples/tree/master/Generic%20Trojan).
This is an unpacked sample which can directly be imported into IDA and the reversing process can be started right away. Upon Importing it IDA automatically detects the `Winmain` function. The first two calls we see on WinMain are `WSAStartup` and `GetModuleFileNameA` as the below screenshot shows:

![alt text]({{ site.baseurl }}/images/SMBWorm/getmodulefilename_1.jpg "{{ site.baseurl }}/images/SMBWorm/getmodulefilename_1.jpg")

