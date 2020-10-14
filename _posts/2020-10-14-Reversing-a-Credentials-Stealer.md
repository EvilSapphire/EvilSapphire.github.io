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

Then the malware employs multiple persistence mechanisms such as copying itself into <SYSDIR>\load32.exe file and adding \<SYSDIR>\load32.exe to HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run. It also queries for the 'STARTUP' key in HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\Shell Registry which stores the directory location in which all the startup executables for the current user is located, then copies itself to a file named 'rundllw.exe' in that startup folder. It also copies itself to a file named DLLreg.exe in the Windows Directory, and then \<WINDIR>\DLLreg.exe is added against the key named 'Run' in win.ini file. Finally it copies itself to a file named vxdmgr32.exe in the System Directory, and then adds the value “explorer.exe \<SYSDIR>\vxdmgr32.exe” to the boot section against a key named shell in the file system.ini. These entries can be taken as simple IOCs indicating that the machine under observation has been infected.
Then 3 Strings are initialised in Global Memory which are the filenames (<WINDIR>\Bank.log for keylogs, <WINDIR>\bank1.bmp and Windir\Bank2.bmp for screenshots) where Keyboard Logs/ Screen Captures will be stored.  
  
![alt text]({{ site.baseurl }}/images/CredentialsStealer/9_filestrings.JPG "{{ site.baseurl }}/images/CredentialsStealer/9_filestrings.JPG") 

The next code of interest is a call to CreateThread:
![alt text]({{ site.baseurl }}/images/CredentialsStealer/10_CreateThread.JPG "{{ site.baseurl }}/images/CredentialsStealer/10_CreateThread.JPG")

The EntryPoint of this Thread is a Function I labelled as SpawnClipBoardHookWindow in IDA. Taking a peek into the function shows us:
![alt text]({{ site.baseurl }}/images/CredentialsStealer/11_threadentry.JPG "{{ site.baseurl }}/images/CredentialsStealer/11_threadentry.JPG")

A Window being registered, created and its Message Loop. The Callback Window Procedure when this Window receives a message is the function I labeled as ClipBoardHookWndProc
![alt text]({{ site.baseurl }}/images/CredentialsStealer/12_Wndproc.JPG "{{ site.baseurl }}/images/CredentialsStealer/12_Wndproc.JPG")

Taking a look inside this function:
![alt text]({{ site.baseurl }}/images/CredentialsStealer/13_WndprocIntrnl.JPG "{{ site.baseurl }}/images/CredentialsStealer/13_WndprocIntrnl.JPG")

We see that when this WndProc receives the WM_CREATE notification which is during the Window Creation, It calls the `SetClipboardViewer` API. According to MSDN: 
>SetClipboardViewer Adds the specified window to the chain of clipboard viewers. Clipboard viewer windows receive a WM_DRAWCLIPBOARD message whenever the content of the clipboard changes.

Therefore upon creation this window is inserted in the clipboard viewer chain.
For any subsequent calls to this Window procedure, if Msg is equal to 0x308 which is the value for the `WM_DRAWCLIPBOARD` constant as #defined in WinUser.H, which means whenever the content of the clipboard has changed according to MSDN above, it calls :
![alt text]({{ site.baseurl }}/images/CredentialsStealer/14_wmdrawclipboard.JPG "{{ site.baseurl }}/images/CredentialsStealer/14_wmdrawclipboard.JPG")

`OpenClipBoard` to open the clipboard relevant with the current task, and then `GetClipBoardData` to get a Handle to the Clipboard Data in `CF_TEXT` format. Then a string is initialised with the value <Windir>\rundllx.sys.

![alt text]({{ site.baseurl }}/images/CredentialsStealer/15_clipstealer.JPG "{{ site.baseurl }}/images/CredentialsStealer/15_clipstealer.JPG")

The \<Windir>\rundllx.sys is passed to a function that acts as a wrapper over a `CreateFile` call, this creates a file rundllx.sys in the Windows Directory:
![alt text]({{ site.baseurl }}/images/CredentialsStealer/16_cliplogcreate.JPG "{{ site.baseurl }}/images/CredentialsStealer/16_cliplogcreate.JPG")

Then the handle to this File and the handle to the clipboard data in `CF_TEXT` format is passed to a function acting as a wrapper over a `WriteFile` call, which writes the clipboard data to the \<Windir>\rundllx.sys. So this window copies all clipboard data in `CF_TEXT` format and logs them to a file.
![alt text]({{ site.baseurl }}/images/CredentialsStealer/17_cliplogwrite.JPG "{{ site.baseurl }}/images/CredentialsStealer/17_cliplogwrite.JPG")

Next code of interest is:
![alt text]({{ site.baseurl }}/images/CredentialsStealer/18_sock64create.JPG "{{ site.baseurl }}/images/CredentialsStealer/18_sock64create.JPG")

Which creates a file WINDIR\sock64.dll and writes the second PE (hooks.dll) inside the
unpacked PE there. Then:
![alt text]({{ site.baseurl }}/images/CredentialsStealer/19_loadlibGetProc.JPG "{{ site.baseurl }}/images/CredentialsStealer/19_loadlibGetProc.JPG")

It loads the DLL into its Process Address Space via a call to `LoadLibrary` and receives the Handle to the DLL’s two exports `h_Init` and `h_Release` via calls to `GetProcAddress`. If we fire up x32dbg and look at this Loaded DLL in the process space:
(entry in progress)
