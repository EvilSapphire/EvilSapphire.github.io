---
layout: post
title: SMB Worm Writeup (Part 1)
---

This is going to be a write-up on reversing a Malware listed as a Generic Trojan in FabriMagic72's Github Malware sample repo found [here](https://github.com/fabrimagic72/malware-samples). The specific sample is the first one found in the 'Generic Trojan' [folder](https://github.com/fabrimagic72/malware-samples/tree/master/Generic%20Trojan).
This is an unpacked sample which can directly be imported into IDA and the reversing process can be started right away. Upon Importing it IDA automatically detects the `Winmain` function. The first two calls we see on WinMain are `WSAStartup` and `GetModuleFileNameA` as the below screenshot shows:

![alt text]({{ site.baseurl }}/images/SMBWorm/getmodulefilename_1.JPG "{{ site.baseurl }}/images/SMBWorm/getmodulefilename_1.JPG")

`WSAStartup` is the API to start up Windows Socket functionality, which is a common API to call for any Windows executable implementing networking capabilities. `GetModuleFileNameA` retrieves the name of the current executable, which is the name of the malware sample in question, and stores the string to the memory location labelled by `ExistingFileName` by IDA. In the following String assembly operations `repne movsd` and `repne movsb` are used to copy this `ExistingFileName` string to a local string on the stack labelled as `String` by IDA, then ' /SYNC' is concatenated to `String`:

![alt text]({{ site.baseurl }}/images/SMBWorm/existingfilename_strcpy_2.JPG "{{ site.baseurl }}/images/SMBWorm/existingfilename_strcpy_2.JPG")
![alt text]({{ site.baseurl }}/images/SMBWorm/strcat_sync_3.JPG "{{ site.baseurl }}/images/SMBWorm/strcat_sync_3.JPG")

Next we see subsequent calls to `RegCreateKeyExA`, `RegSetValueExA` and `RegCloseKey`.

![alt text]({{ site.baseurl }}/images/SMBWorm/registry_3.JPG "{{ site.baseurl }}/images/SMBWorm/registry_3.JPG")

These calls to the Windows Registry APIs open up the Registry Key "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Run", and to it adds a value named PHIME2008 and stores the concatenated `String` and ' /SYNC' in that value. HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Run is very popular among malwares as this is the Registry key where the paths of Windows Startup executables are stored. So basically this malware is trying to make itself persistent by storing the name of its executable to this registry key so that every time the machine starts this piece of malware will automatically be launched.

After these there are two subsequent calls to sub_401EB0 and sub_401C40.
![alt text]({{ site.baseurl }}/images/SMBWorm/twocalls_5.JPG "{{ site.baseurl }}/images/SMBWorm/twocalls_5.JPG")

A peek into sub_401eb0 gives us the first 3 calls to `GetSystemDirectoryA`, `strcat` and `lopen`.
![alt text]({{ site.baseurl }}/images/SMBWorm/sysdir_6.JPG "{{ site.baseurl }}/images/SMBWorm/sysdir_6.JPG")



The malware stores the System Directory location (C:\Windows\System32) to a string, appends 'msupd.exe' to it checks if the 'C:\Windows\System32\msupd.exe' exists if by trying to open it with `lopen`. In case it doesn't, as the subsequent calls show us that

![alt text]({{ site.baseurl }}/images/SMBWorm/deleteurlcachedownload_7.JPG "{{ site.baseurl }}/images/SMBWorm/deleteurlcachedownload_7.JPG")

![alt text]({{ site.baseurl }}/images/SMBWorm/urldownloadtofile_8.JPG "{{ site.baseurl }}/images/SMBWorm/urldownloadtofile_8.JPG")

it tries to download http: //fukyu<.>jp/updata/ACCl3.jpg (**!WARNING!**) 's content to the C:\Windows\System32\msupd.exe file, and then tries to run the executable with a call to `CreateProcessA`.

![alt text]({{ site.baseurl }}/images/SMBWorm/9_createprocess.JPG "{{ site.baseurl }}/images/SMBWorm/9_createprocess.JPG")

The jpg file on the malicious website is then actually a windows PE executable file. At the time of writing this the domain doesn't have a DNS record therefore has been taken down, so we can't go ahead with analysing its content. It is likely that the file was identical to this same worm that we are reversing therefore launching multiple copies of itself, however there is no way to be certain. Be that as it may, we can go forward and analyse the rest of the malware.

After the call to sub_401eb0, there's another call as seen to sub_401c40. Let's take a look at what this subroutine does:

![alt text]({{ site.baseurl }}/images/SMBWorm/13_connect.JPG "{{ site.baseurl }}/images/SMBWorm/13_connect.JPG")

First there's a call to `connect` to the malware's C2C server 125[.]206[.]117[.]59 over TCP port 80 (**!WARNING!**). This was the DNS record of fukyu[.]jp (**!WARNING!**) when the domain was alive. If the call to `connect` succeeds, the malware goes on and collects various user info from the infected host and places them to multiple strings on the stack:

![alt text]({{ site.baseurl }}/images/SMBWorm/10_collectdata.JPG "{{ site.baseurl }}/images/SMBWorm/10_collectdata.JPG")

The APIs `GetLocaleInfoA` and `GetComputerNameA` are self explanatory. A peek into the other two subroutines sub_401A70 and sub_402090 reveals that the first subroutine:

![alt text]({{ site.baseurl }}/images/SMBWorm/14_getsystime.JPG "{{ site.baseurl }}/images/SMBWorm/14_getsystime.JPG")

Gets the system time at which the malware is running and places the information to a string. And sub_402090:

![alt text]({{ site.baseurl }}/images/SMBWorm/11_getlocalIP.JPG "{{ site.baseurl }}/images/SMBWorm/11_getlocalIP.JPG")

Gets the Local IP of the system via two calls to `gethostname` and `gethostbyname` and enumerating the returned `hostent` structure.

This collected information is placed to a HTTP GET Request string using the `_sprintf` call below and sent over to fukyu[.]jp/updata/TPDB[.]php (**!WARNING!**) via a HTTP request. The functionality on this PHP page must have been to collect the data sent via the GET request and store them in the database, so that the C2C server of the malware will have a record of all the infected hosts and the time at which they were infected.

![alt text]({{ site.baseurl }}/images/SMBWorm/12_send.JPG "{{ site.baseurl }}/images/SMBWorm/12_send.JPG")

After collecting the infected user data and sending it to the C2C server, there's a loop that iterates 100 times and each time it launches a new thread with the CRT thread creation function `beginthreadex`. The subroutine that is the entrypoint of these threads are sub_401870 as seen from the below screenshot. Therefore sub_401870 must be where the functionality of the spread of the worm is implemented.
![alt text]({{ site.baseurl }}/images/SMBWorm/15_beginthreadex.JPG "{{ site.baseurl }}/images/SMBWorm/15_beginthreadex.JPG")

In sub_401870 the first interesting code is this little do while loop that calls sub_401140 four times. 

![alt text]({{ site.baseurl }}/images/SMBWorm/19-randoutside.JPG "{{ site.baseurl }}/images/SMBWorm/19-randoutside.JPG") 

This sub_401140 is nothing but a wrapper over a call to `rand` which generates a random byte DWORD value, shifts the value to the right by 1 byte and returns it to the loop.
This DWORD is stored to a variable on the stack. This is done four times.
![alt text]({{ site.baseurl }}/images/SMBWorm/17_randint.JPG "{{ site.baseurl }}/images/SMBWorm/17_randint.JPG")

These 4 random DWORDs are then ANDed with 0xFF to extract the least significant byte and passed to a `_sprintf` call which takes in a format string of the form `"%d.%d.%d.%d"`. Which means these random bytes are placed in four octet IP format (e.g 8.9.10.11) in a string and this string is passed to sub_401150 the return value of which is checked to determine whether the loop will terminate. Taking a look at sub_401150 gives us:

![alt text]({{ site.baseurl }}/images/SMBWorm/18-privateIPChecker.JPG "{{ site.baseurl }}/images/SMBWorm/18-privateIPChecker.JPG")

This very obvious branching logic that checks for different octets in this IP string whether they are of the format '10.x.x.x' / '172.16-31.x.x' / '192.168.x.x'/ '127.x.x.x'. In case they are, the function returns a value of 1. If the function returns a non-zero value, the outer do-while loop does not terminate and generates a new set of random values. These are formats of private IP subnets, therefore it's clear sub_401150 is checking whether the randomly generated IP is a private IP and in case it is, it generates a new random IP. Therefore the worm will try to connect to public IPs only.

After this Public IP Check passes, there's another call to sub_4011F0 with the public IP string as argument, the return value of which is also checked to see if the loop terminates. Peeking into this subroutine gives us:

![alt text]({{ site.baseurl }}/images/SMBWorm/20_telnet.JPG "{{ site.baseurl }}/images/SMBWorm/20_telnet.JPG")

In this subroutine first there's a call to `socket` which creates a TCP socket, then this socket is set to non-blocking mode via the call to `ioctlsocket`. The handle to this socket is passed to a `connect` call which also takes in the argument of a `sockaddr` structure to which the IP value fed is the randomly generated public IP string, and the port is set to be 445. Therefore this `connect` call initiates a TCP handshake towards <Public IP>:445. The call to `select` WINAPI down below checks the writability of this socket and if this call succeeds, the function returns 1 and the outer loop terminates. So basically this subroutines initiaties a TCP connection towards the Public IP on port 445 and checks if the connection attempt has completed and if the socket is writable. Therefore we can assume data will be sent towards the public IP on port 445. We are seeing the first signs that this malware is actually an SMB worm.
  
After this there's a large do while loop:

![alt text]({{ site.baseurl }}/images/SMBWorm/21_for1.JPG "{{ site.baseurl }}/images/SMBWorm/21_for1.JPG")
![alt text]({{ site.baseurl }}/images/SMBWorm/22_for2.JPG "{{ site.baseurl }}/images/SMBWorm/22_for2.JPG")

The assembly may not look that readable, but what this loop is doing is that it removes the last octet of the generated public IP string, appends 1 to it, and passes the string to the same sub_4011F0 connectibility check subroutine described above. If the connectivity check passes, the IP is formatted into a string format '\\\\\<public-ip>' using `sprintf` and passed to sub_4012B0. Then the last octet is set 2 and the same thing repeats, and the loop continues until the last octet is less than 254. Therefore the entire /24 subnet of the randomly generated public IP is checked for socket connectivity and subsequently the '\\\\\<public-IP>' string is passed to sub_4012B0. This \\\\\<public-IP> is obviously the format used to access Windows SMB shares, so sub_4012B0 must be using this string to enumerate and possible send data to the Windows SMB share for which the connectivity check passed. sub_4012B0 is actually the meat of the worm functionality, and therefore its discussion warrants a separate post. The first part of the analysis ends here, if you read this, I would happy to know your thoughts if you would spare some of your time. I can be reached at github or roysomik@yahoo.com/ evilsapphire_s@yahoo.in. Thank you for reading!


  
