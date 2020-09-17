---
layout: post
title: SMB Worm Writeup
---

This is going to be a write-up on reversing a Malware listed as a Generic Trojan in FabriMagic72's Github Malware sample repo found [here](https://github.com/fabrimagic72/malware-samples). The specific sample is the first one found in the 'Generic Trojan' [folder](https://github.com/fabrimagic72/malware-samples/tree/master/Generic%20Trojan).
This is an unpacked sample which can directly be imported into IDA and the reversing process can be started right away. Upon Importing it IDA automatically detects the `Winmain` function. The first two calls we see on WinMain are `WSAStartup` and `GetModuleFileNameA` as the below screenshot shows:

![alt text]({{ site.baseurl }}/images/SMBWorm/getmodulefilename_1.JPG "{{ site.baseurl }}/images/SMBWorm/getmodulefilename_1.JPG")

`WSAStartup` is the API to start up Windows Socket functionality. `GetModuleFileNameA` retrieves the name of the current executable, which is the malware sample, and stores the string to the memory offset in the PE labelled by `ExistingFileName` by IDA. In the following String assembly operations `repne movsd` and `repne movsb` are used to copy this `ExistingFileName` string to a local string on the stack labelled as `String` by IDA, the '/SYNC' is concatenated to `String`:

![alt text]({{ site.baseurl }}/images/SMBWorm/existingfilename_strcpy_2.JPG "{{ site.baseurl }}/images/SMBWorm/existingfilename_strcpy_2.JPG")
![alt text]({{ site.baseurl }}/images/SMBWorm/strcat_sync_3.JPG "{{ site.baseurl }}/images/SMBWorm/strcat_sync_3.JPG")

Next we see subsequent calls to `RegCreateKeyExA`, `RegSetValueExA` and `RegCloseKey`.

![alt text]({{ site.baseurl }}/images/SMBWorm/registry_3.JPG "{{ site.baseurl }}/images/SMBWorm/registry_3.JPG")



