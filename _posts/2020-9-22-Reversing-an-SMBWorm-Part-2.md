---
layout: post
title: Write-up on Reversing an SMB Worm (Part 2)
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

Next there's a call to `CopyFileA` two which the arguments passed are the string holding the '\\\\\<Public-IP>\\\\admin$\\\\system32\\\\dnsapi.exe' value and the `ExistingFileName` string to which the current executable fully qualified path name was placed back in the `WinMain` function (refer to part 1). The '\\\\\<Public-IP>\\\\admin$' part accesses the administrator share of the remote server, which basically resolves to the 'C:\\Windows' folder of the Remote server, therefore the `CopyFileA` call copies the content of the currently executing malware to a file called dnsapi.exe existing on the C:\\Windows\\system32 folder of the Remote Server. This is basically how the worm propagates itself.

![alt text]({{ site.baseurl }}/images/SMBWorm/33_copyfile.JPG "{{ site.baseurl }}/images/SMBWorm/33_copyfile.JPG")

After the malware's content has been successfully copied over to the remote server, a call to sub_401A70 is made with pointer to a local string passed to it as the parameter. 
![alt text]({{ site.baseurl }}/images/SMBWorm/32_infectedtime.JPG "{{ site.baseurl }}/images/SMBWorm/32_infectedtime.JPG")

Taking a look at the sub_401A70 we find that it gets the System Local time via a `GetLocalTime` call and inserts the hour, minute, second values to the argument string via a call to `sprintf`. Therefore `sub_401A70` basically retrieves the time at which it has successfully been able to copy the file to the remote server to a local string on the stack.

![alt text]({{ site.baseurl }}/images/SMBWorm/35_401a70.JPG "{{ site.baseurl }}/images/SMBWorm/35_401a70.JPG")

Next, a call to `sub_401AE0` is made, and the arguments passed to it are the string containing the local time retrieved by `sub_401A70` above, a string 'cp', the Public IP of the remote server, and the Username and Password that were successfully authenticated on remote server. 

![alt text]({{ site.baseurl }}/images/SMBWorm/40_c2ccopyok.JPG "![alt text]({{ site.baseurl }}/images/SMBWorm/40_c2ccopyok.JPG")

Taking a peek inside the `sub_401A70` function shows us:
![alt text]({{ site.baseurl }}/images/SMBWorm/41_c2ccopyokconnect.JPG "{{ site.baseurl }}/images/SMBWorm/41_c2ccopyokconnect.JPG")

The malware's C2C server 125[.]206[.]117[.]59 is connected to, and a HTTP GET Request is sent to a PHP page /update/TPDA.php hosted on this C2C with  multiple HTTP parameters set to the values of the remote server's Public IP, authenticated Username and Password. The 'cp' string must be used to indicate to the server that the COPYING of the malware's content to dnsapi.exe on the remote server has successfully completed. So this function basically communicates to the C2C the public IP, username, password for which a successful authentication/copy has completed.

![alt text]({{ site.baseurl }}/images/SMBWorm/42_c2ccopyoksend.JPG "{{ site.baseurl }}/images/SMBWorm/42_c2ccopyoksend.JPG")

After the C2C is informed about the Copy OK action, there's a call to `NetRemoteTOD` API that retrieves the local time of the Remote Server to a `TIME_OF_DAY_INFO` structure pointed to by `BufferPtr`. 
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
It is clear the `tod_minutes`,  `tod_timezone` and `tod_hours` values are being moved to these register. Then some assembly mathematical operation is performed to calculate the value of `abs(BufferPtr->tod_timezone) + 60 * BufferPtr->tod_hours + BufferPtr->tod_mins + 2` and place it into the `eax` register. This time value is clearly 2 minutes ahead of the current time of the Remote Server.
![alt text]({{ site.baseurl }}/images/SMBWorm/38_bufferptrcalc.JPG "{{ site.baseurl }}/images/SMBWorm/38_bufferptrcalc.JPG")

An `AT_INFO` structure is then initialised on the stack. The calculated time is passed to the `JobTime` member of the struct, and a string `dnsapi.exe` is passed to its `Command` member. This `AT_INFO` struct is then passed to a `NetScheduleJobAdd` API which schedules the dnsapi.exe file on the remote server to run 2 minutes after the remote server's current time. As seen earlier the malware's code has already been copied to the dnsapi.exe file, therefore the malware is scheduled to run on the server after 2 minutes. This is clearly worm like behaviour where it copies itself onto a remote server and schedules itself to run after a fixed time.

![alt text]({{ site.baseurl }}/images/SMBWorm/39_netschedulejobadd.JPG "{{ site.baseurl }}/images/SMBWorm/39_netschedulejobadd.JPG")

In case the call to `NetScheduleJobAdd` fails, the dnsapi.exe on the remote server is deleted via a call to `DeleteFileA` 
![alt text]({{ site.baseurl }}/images/SMBWorm/43_deletefilea.JPG "{{ site.baseurl }}/images/SMBWorm/43_deletefilea.JPG")

In case the call to `NetScheduleJobAdd` succeeds and the execution of the worm is successfully scheduled on the remote server, we see another call to `sub_401AE0` as above when the copying action was completed, but this time the time at which the scheduling was completed is passed as an argument with the string 'at', which makes it clear this call basically informs the C2C that the task of scheduling the execution of the worm on the remote server has successfully completed as well.
![alt text]({{ site.baseurl }}/images/SMBWorm/44_taskok.JPG "{{ site.baseurl }}/images/SMBWorm/44_taskok.JPG")

And this pretty much concludes the analysis of the inner-workings of this worm. In a nutshell, first the worm creates a entry in the HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Run Registry location with the name PHIME2008 and with fully qualified current executable name as its value, which makes the malware persistent by making it a startup entry in the infected machine. Then it downloads a PE file to the infected machine's System32 folder named as msupd.exe which is likely the same worm sample and then runs it. The worm then collects some data about the infected local machine and sends the data over to its C2C server. It then generates random Public IPs, sees if a TCP connection to the remote public IP on port 445 can be made, and in case the connection succeeds again checks if the IPC$ SMB session on the remote IP can be accessed. In case it can be the worm calls `NetUserEnum` to enumerate the user list on the remote server, then for each user a commonly used password list is read and each of the username/password combination is tried for authentication. In case a username/password combination succeeds for authentication, the malware's content is copied to C:\\Windows\\System32\\dnsapi.exe file on the remote server. The C2C is informed about the successful copy action, then dnsapi.exe on the remote server is scheduled to run 2 minutes after the current time of the remote server. The C2C is also informed about the successful scheduling action. Thereby propagation of the worm over SMB shares is achieved.

That's it for this malware sample. Please let me know what you think and how the write-up can be improved. I can be contacted on the e-mails roysomik@yahoo.com/ evilsapphire_s@yahoo.in. For now, mata ne!
 
