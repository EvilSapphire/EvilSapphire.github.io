---
layout: post
title: SMB Worm Writeup (Part 2)
---

Welcome to the part 2 of reversing an SMB Worm. In [part 1](https://evilsapphire.github.io/SMBWorm-Writeup-Part-1/) we observed the worm generating random Public IPs, then checking whether data can be sent to over to the Public via calls to `connect` and `select` WINAPIs, and if the check succeded it generated the SMB Share format of the IP by appending '\\\\' before the Public IP string and passed this '\\\\\<Public-IP>' to `sub_4012B0`. In this post we are going to take a look at how exactly the worm is propagating itself over SMB connections.
The first thing `sub_4012B0` does is that it appends the string 'ipc$' after the '\\\\\<Public-IP>' string via a call to `sprintf`. This string is passed to the `WNetAddConnection2A` API via setting the IPC string to the `NetResource.lpRemoteName` attribute. 
![alt text]({{ site.baseurl }}/images/SMBWorm/23_ipcstr.JPG "{{ site.baseurl }}/images/SMBWorm/23_ipcstr.JPG")
The IPC(Inter-process communication) share is a null session connection that exists on Windows that lets anonymous users enumerate some limited properties of the shared machine. From the MSDN documentation:
>The IPC$ share is also known as a null session connection. By using this session, Windows lets anonymous users perform certain activities, such as enumerating the names of domain >accounts and network shares. The IPC$ share is created by the Windows Server service. This special share exists to allow for subsequent named pipe connections to the server.
Therefore the function checks whether a connection to the null session on the Public IP can be made. In case the call to `WNetAddConnection2A` fails with an error code, the function exits.
![alt text]({{ site.baseurl }}/images/SMBWorm/24_ipcwnet.JPG "{{ site.baseurl }}/images/SMBWorm/24_ipcwnet.JPG")
If call to `WNetAddConnection2A` succeeds, the '\\\\\<Public-IP>' string is passed to a `NetUserEnum` call. The `NetUserEnum` API as its name indicates enumerates all the user accounts on a server and returns the information to an array of `USER_INFO` structure (`bufptr` in the arguments labelled by IDA).
![alt text]({{ site.baseurl }}/images/SMBWorm/25_netuserenum.JPG "{{ site.baseurl }}/images/SMBWorm/25_netuserenum.JPG")
If the call to `NetUserEnum` succeeds, the pointer to this array of `USER_INFO_0` structures `bufptr` is moved to the `esi` register ![alt text]({{ site.baseurl }}/images/SMBWorm/26-bufptrmove.JPG "{{ site.baseurl }}/images/SMBWorm/26-bufptrmove.JPG"), and then each of these `USER_INFO_0` structure is enumerated inside a for loop until a counter reaches the number of entries read from the server which is also returned by the `NetUserEnum` API.
![alt text]({{ site.baseurl }}/images/SMBWorm/27_401430loop.JPG "{{ site.baseurl }}/images/SMBWorm/27_401430loop.JPG")
 


 
