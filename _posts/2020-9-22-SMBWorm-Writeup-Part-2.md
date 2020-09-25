---
layout: post
title: SMB Worm Writeup (Part 2)
---

Welcome to the part 2 of reversing an SMB Worm. In [part 1](https://evilsapphire.github.io/SMBWorm-Writeup-Part-1/) we observed the worm generating random Public IPs, then checking whether data can be sent to over to the Public IP via calls to `connect` and `select` WINAPIs, and if the check succeded it generated the SMB Share format of the Public IP by appending '\\\\' before the IP string and passed this '\\\\\<Public-IP>' string to `sub_4012B0`. In this post we are going to take a look at how exactly the worm is propagating itself over SMB connections.

#### sub_4012B0:

The first thing `sub_4012B0` does is that it appends the string 'ipc$' after the '\\\\\<Public-IP>' string via a call to `sprintf`. This string is passed to the `WNetAddConnection2A` API via setting the IPC string to the `NetResource.lpRemoteName` attribute. Therefore this initiates a connection to the IPC SMB session of the server with the public IP.

![alt text]({{ site.baseurl }}/images/SMBWorm/23_ipcstr.JPG "{{ site.baseurl }}/images/SMBWorm/23_ipcstr.JPG")

The IPC(Inter-process communication) share is a null session connection that exists on Windows that lets anonymous users enumerate some limited properties of the shared machine. From the MSDN documentation:
>The IPC$ share is also known as a null session connection. By using this session, Windows lets anonymous users perform certain activities, such as enumerating the names of domain accounts and network shares. The IPC$ share is created by the Windows Server service. This special share exists to allow for subsequent named pipe connections to the server.

Therefore the function checks whether a connection to the null session on the Public IP can be made, and therefore server properties over the SMB connection can be enumerated. In case the call to `WNetAddConnection2A` fails with an error code, the function exits.

![alt text]({{ site.baseurl }}/images/SMBWorm/24_ipcwnet.JPG "{{ site.baseurl }}/images/SMBWorm/24_ipcwnet.JPG")

If call to `WNetAddConnection2A` succeeds, the '\\\\\<Public-IP>' string is passed to a `NetUserEnum` call. The `NetUserEnum` API, as its name indicates, enumerates all the user accounts on a server and returns the information to an array of `USER_INFO` structures (`bufptr` in the arguments labelled by IDA). 

![alt text]({{ site.baseurl }}/images/SMBWorm/25_netuserenum.JPG "{{ site.baseurl }}/images/SMBWorm/25_netuserenum.JPG")

In this case, the `level` argument passed to `NetUserEnum` is 0, therefore the returned `bufptr` will be in the form of the structure `USER_INFO_0`, which has the following struct definition:
```
typedef struct _USER_INFO_0 {
  LPWSTR usri0_name;
} USER_INFO_0, *PUSER_INFO_0, *LPUSER_INFO_0;
```
Therefore this `NetUserEnum` API will return an array of structures that contains the name of the users in the server in the `usri0_name` member of the structure.
If the call to `NetUserEnum` succeeds, the pointer to this array of `USER_INFO_0` structures `bufptr` is moved to the `esi` register
![alt text]({{ site.baseurl }}/images/SMBWorm/26-bufptrmove.JPG "{{ site.baseurl }}/images/SMBWorm/26-bufptrmove.JPG"), and then each of these `USER_INFO_0` structure is enumerated inside a for loop until a counter reaches the number of entries read from the server which is also returned by the `NetUserEnum` API.

![alt text]({{ site.baseurl }}/images/SMBWorm/27_401430loop.JPG "{{ site.baseurl }}/images/SMBWorm/27_401430loop.JPG")
 
Clearly, in each iteration of the loop, the `bufptr->usri0_name` string is retrieved and passed to `sub_401430` function. `sub_401430` also takes in the '\\\\\<Public-IP>' string as an argument. Therefore in summary what `sub_4012B0` does is that it enumerates the public IP for all configured user accounts, and passes each of those user account to `sub_401430` for further processing. Then we need to take a look at `sub_401430`.

#### sub_401430:

`sub_401430` is a fairly small function where first the memory address 0x408030 is moved into the esi register.

![alt text]({{ site.baseurl }}/images/SMBWorm/28_401490loop.JPG "{{ site.baseurl }}/images/SMBWorm/28_401490loop.JPG")

If we take a look at the content at this offset in IDA we find what readily appears to be a list of commonly used passwords.

![alt text]({{ site.baseurl }}/images/SMBWorm/29_passwordlist.JPG "{{ site.baseurl }}/images/SMBWorm/29_passwordlist.JPG")

In the do while loop starting from address 0x40145C, clearly each of these passwords are being read, and then passed to a function `sub_401490` along with the '\\\\\<Public-IP>' string and the username that came as the argument to `sub_401430`. Therefore `sub_401430` reads the entire commonly used password list and passes all the passwords for each username to `sub_401490`. Therefore `sub_401490` must be using these username/password pair and trying them to gain access to the server somehow. Let us take a look at it next.

#### sub_401490:

Inside `sub_401490`, we see the access gaining attempt right away, where there's a call to `WNetAddConnection2A` with the `lpUserName` and `lpPassword` arguments set to the Username and Password passed to `sub_401490`.

![alt text]({{ site.baseurl }}/images/SMBWorm/30_unamepassconnect.JPG "{{ site.baseurl }}/images/SMBWorm/30_unamepassconnect.JPG").

If this call to `WNetAddConnection2A` succeeds, which means the worm has successfully authenticated and gained access to the server with this username/password combination, a string is initialised with the value '\\\\\<Public-IP>\\\\admin$\\\\system32\\\\dnsapi.exe'.   
![alt text]({{ site.baseurl }}/images/SMBWorm/31_SysdirDNSApiString.JPG "{{ site.baseurl }}/images/SMBWorm/31_SysdirDNSApiString.JPG")

Next there is a call to sub_401A70 with pointer to a local string passed to it as the parameter. 
![alt text]({{ site.baseurl }}/images/SMBWorm/32_infectedtime.JPG "{{ site.baseurl }}/images/SMBWorm/32_infectedtime.JPG")

Taking a look at the sub_401A70 we find that it gets the System Local time via a `GetLocalTime` call and inserts the hour, minute, second values to the argument string via a call to `sprintf`. Therefore `sub_401A70` basically retrieves the time at which it has successfully been able to login to the remote server to a local string on the stack.

![alt text]({{ site.baseurl }}/images/SMBWorm/35_401a70.JPG "{{ site.baseurl }}/images/SMBWorm/35_401a70.JPG")

Next there's a call to `CopyFileA` two which the arguments passed are the string holding the '\\\\\<Public-IP>\\\\admin$\\\\system32\\\\dnsapi.exe' value and the `ExistingFileName` string to which the current executable fully qualified path name was placed back in the `WinMain` function (refer to part 1). The '\\\\\<Public-IP>\\\\admin$' part accesses the administrator share of the remote server, which basically resolves to the 'C:\\Windows' folder of the Remote server, therefore the `CopyFileA` call copies the content of the currently executing malware to a file called dnsapi.exe existing on the C:\\Windows\\system32 folder of the Remote Server. This is basically how the worm propagates itself.

![alt text]({{ site.baseurl }}/images/SMBWorm/33_copyfile.JPG "{{ site.baseurl }}/images/SMBWorm/33_copyfile.JPG")

After the malware's content has been copied over to the remote server, there's a call to `NetRemoteTOD` API that retrieves the local time of the Remote Server to a `TIME_OF_DAY_INFO` structure pointed to by `BufferPtr`. 
![alt text]({{ site.baseurl }}/images/SMBWorm/36_NetremoteTOD.JPG "{{ site.baseurl }}/images/SMBWorm/36_NetremoteTOD.JPG")

This `BufferPtr` pointer is moved to `ecx` as per the following screenshot, and then different members residing in different offset in this structure is moved to the registers `edi`, `eax` and `ecx`.
![alt text]({{ site.baseurl }}/images/SMBWorm/37_bufferptrNetRemoteTOD.JPG "{{ site.baseurl }}/images/SMBWorm/37_bufferptrNetRemoteTOD.JPG")

Taking a look at the `TIME_OF_DAY_INFO` struct definition:
```
typedef struct _TIME_OF_DAY_INFO {
  DWORD tod_elapsedt;
  DWORD tod_msecs;
  DWORD tod_hours;
  DWORD tod_mins;
  DWORD tod_secs;
  DWORD tod_hunds;
  LONG  tod_timezone;
  DWORD tod_tinterval;
  DWORD tod_day;
  DWORD tod_month;
  DWORD tod_year;
  DWORD tod_weekday;
} TIME_OF_DAY_INFO, *PTIME_OF_DAY_INFO, *LPTIME_OF_DAY_INFO;
```
It is clear the `tod_minutes`,  `tod_timezone` and `tod_hours` values are being moved to these register. Then some assembly mathematical operation is performed to calculate the value of `abs(BufferPtr->tod_timezone)+ 60 * BufferPtr->tod_hours+ BufferPtr->tod_mins+ 2` and place it into the `eax` register. This time value is clearly 2 minutes ahead of the current time of the Remote Server.
![alt text]({{ site.baseurl }}/images/SMBWorm/38_bufferptrcalc.JPG "{{ site.baseurl }}/images/SMBWorm/38_bufferptrcalc.JPG")

An `AT_INFO` structure is then initialised on the stack. The calculated time is passed to the `JobTime` member of the struct, and a string `dnsapi.exe` is passed to its `Command` member. This `AT_INFO` struct is then passed to a `NetScheduleJobAdd` API which schedules the dnsapi.exe file on the remote server to run in the 2 minutes after the remote server's current time. As seen earlier the malware's code has already been copied to the dnsapi.exe file, therefore the malware is scheduled to run on the server after 2 minutes. This is clearly worm like behaviour where it copies itself onto a remote server and schedules itself to run after a fixed time.

![alt text]({{ site.baseurl }}/images/SMBWorm/39_netschedulejobadd.JPG "{{ site.baseurl }}/images/SMBWorm/39_netschedulejobadd.JPG)



  




 
