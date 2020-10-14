---
layout: post
title: Reversing a Barclays Credentials Stealer (Part 1)
---

I recently had my hands on a malware sample that came with a malicious e-mail attachment. Upon reversing it I found out it employed various facets of Windows Programming to steal Barclay's Bank credentials from an unsuspecting user. It took advantage of Windows GUI programming flexibility to insert a Window inside the clipboard chain to log user clipboard data. It also extensively used the COM framework to determine the user behaviour and accordingly steal user’s data. The malware definitely warrants a entry in this personal Malware Analysis write-up blog.

The file that I renamed as 'mal.fil' is a simple PE file that can be opened with a PE viewer tool like PE Bear.
![alt text]({{ site.baseurl }}/images/CredentialsStealer/1_file.JPG "{{ site.baseurl }}/images/CredentialsStealer/1_file.JPG")

If we take a look at the sections, we can clearly see this is a file packed with UPX packers as indicated by the sections names UPX0, UPX1.
![alt text]({{ site.baseurl }}/images/CredentialsStealer/2_UPXsections.JPG "{{ site.baseurl }}/images/CredentialsStealer/2_UPXsections.JPG") 

There are multiple ways to unpack a UPX packed file . UPX generally unpacks the original PE and then jumps to the OEP via a JMP instruction. So we can run this on a debugger and wait till the JMP is hit and dump out the unpacked executable from memory. An easier way is to install UPX packer itself that comes with the unpacker flag ‘-d’ and unpack it.
![alt text]({{ site.baseurl }}/images/CredentialsStealer/3_upxunpack.JPG "{{ site.baseurl }}/images/CredentialsStealer/3_upxunpack.JPG")

This gives us the unpacked malware. Taking a look at it on HxD we find there is actually a second PE starting at raw offset 0x2F4A6 or RVA 0x300A6 unpacked by UPX inside this PE.

![alt text]({{ site.baseurl }}/images/CredentialsStealer/4_hxd2.JPG "{{ site.baseurl }}/images/CredentialsStealer/4_hxd2.JPG")

Dumping this PE out and viewing it on PE Bear shows us it is a DLLnamed hooks.dll with two exports h_Init and h_Release:
![alt text]({{ site.baseurl }}/images/CredentialsStealer/5_diskdll.JPG "{{ site.baseurl }}/images/CredentialsStealer/5_diskdll.JPG")

However, the disassembly for these exports looks garbled, because it looks like it is an Aspack packed DLL.
![alt text]({{ site.baseurl }}/images/CredentialsStealer/6_aspackdll.JPG "{{ site.baseurl }}/images/CredentialsStealer/6_aspackdll.JPG")

![alt text]({{ site.baseurl }}/images/CredentialsStealer/7_aspackdll.JPG "{{ site.baseurl }}/images/CredentialsStealer/7_aspackdll.JPG")

As we will see later, DLL exports will be decrypted and the imports will be resolved when the
DLL is actually loaded in memory

Importing the unpacked malware into IDA takes us straight to the WinMain function. First CoInitialize API is called which tells us this malware is likely to use COM framework.
CreateMutex is called to create a mutex called ‘BarclMutex’ and if ERROR_ALREADY_EXISTS is returned, the malware stops execution. So only one instance of the malware can be run at a time.
![alt text]({{ site.baseurl }}/images/CredentialsStealer/8_winmainstart.JPG "{{ site.baseurl }}/images/CredentialsStealer/8_winmainstart.JPG")

Then the malware employs multiple persistence mechanisms such as copying itself into <SYSDIR>\load32.exe file and adding <SYSDIR>\load32.exe to HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run. It also queries for the 'STARTUP' key in HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\Shell Registry which stores the directory location in which all the startup executables for the current user is located, then copies itself to a file named 'rundllw.exe' in that startup folder. It also copies itself to a file named DLLreg.exe in the Windows Directory, and then <WINDIR>\DLLreg.exe is added against the key named 'Run' in win.ini file. Finally it copies itself to a file named vxdmgr32.exe in the System Directory, and then adds the value “explorer.exe <SYSDIR>\vxdmgr32.exe” to the boot section against a key named shell in the file system.ini. These entries can be taken as simple IOCs indicating that the machine under observation has been infected.
Then 3 Strings are initialised in Global Memory which are the filenames (<WINDIR>\Bank.log for keylogs, <WINDIR>\bank1.bmp and Windir\Bank2.bmp for screenshots) where Keyboard Logs/ Screen Captures will be stored.  
  
![alt text]({{ site.baseurl }}/images/CredentialsStealer/9_filestrings.JPG "{{ site.baseurl }}/images/CredentialsStealer/9_filestrings.JPG") 
  
(entry in progress)
