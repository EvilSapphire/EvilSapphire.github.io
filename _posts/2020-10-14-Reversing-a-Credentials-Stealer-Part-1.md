---
layout: post
title: Reversing a Barclays Bank Credentials Stealer (Part 1)
---

I recently had my hands on a malware sample that came with a malicious e-mail attachment. Upon reversing it I found out it employed various facets of Windows Programming to steal the Barclays Bank credentials from an unsuspecting user. It took advantage of the flexibility Windows GUI programming offers to insert a Window inside the clipboard viewer chain to log user clipboard data. It also extensively used the COM framework to determine the user behaviour and accordingly steal user’s data. The malware definitely warrants a entry in this Malware Analysis write-up blog.

The file that I renamed as 'mal.fil' is a simple PE file that can be opened with a PE viewer tool like PE Bear.
![alt text]({{ site.baseurl }}/images/CredentialsStealer/1_file.JPG "{{ site.baseurl }}/images/CredentialsStealer/1_file.JPG")

If we take a look at the sections, we can clearly see this is a file packed with UPX packer as indicated by the sections names UPX0, UPX1.
![alt text]({{ site.baseurl }}/images/CredentialsStealer/2_UPXsections.JPG "{{ site.baseurl }}/images/CredentialsStealer/2_UPXsections.JPG") 

There are multiple ways to unpack a UPX packed file . UPX generally unpacks the original PE and then jumps to the Original Entry Point via a JMP instruction. So we can run this on a debugger and wait till the JMP is hit and dump out the unpacked executable from memory. An easier way is to install UPX packer itself that comes with the unpacker flag ‘-d’ and unpack it.

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

Then the malware employs multiple persistence mechanisms such as copying itself into \<SYSDIR>\\load32.exe file and adding \<SYSDIR>\\load32.exe to HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run. It also queries for the 'STARTUP' key in HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Explorer\Shell Registry which stores the directory location in which all the startup executables for the current user is located, then copies itself to a file named 'rundllw.exe' in that startup folder. It also copies itself to a file named DLLreg.exe in the Windows Directory, and then \<WINDIR>\DLLreg.exe is added against the key named 'Run' in win.ini file. Finally it copies itself to a file named vxdmgr32.exe in the System Directory, and then adds the value “explorer.exe \<SYSDIR>\vxdmgr32.exe” to the boot section against a key named shell in the file system.ini. These entries can be taken as simple IOCs indicating that the machine under observation has been infected.
Then 3 Strings are initialised in Global Memory which are the filenames (\<WINDIR>\Bank.log for keylogs, \<WINDIR>\bank1.bmp and \<WINDIR>\Bank2.bmp for screenshots) where Keyboard Logs/ Screen Captures will be stored.  
  
![alt text]({{ site.baseurl }}/images/CredentialsStealer/9_filestrings.JPG "{{ site.baseurl }}/images/CredentialsStealer/9_filestrings.JPG") 

The next code of interest is a call to CreateThread:

![alt text]({{ site.baseurl }}/images/CredentialsStealer/10_CreateThread.JPG "{{ site.baseurl }}/images/CredentialsStealer/10_CreateThread.JPG")

The EntryPoint of this Thread is a Function I labelled as SpawnClipBoardHookWindow in IDA. Taking a peek into the function shows us:
![alt text]({{ site.baseurl }}/images/CredentialsStealer/11_threadentry.JPG "{{ site.baseurl }}/images/CredentialsStealer/11_threadentry.JPG")

A Window being registered, created and its Message Loop. The Callback Window Procedure when this Window receives a message is the function I labeled as ClipBoardHookWndProc

![alt text]({{ site.baseurl }}/images/CredentialsStealer/12_Wndproc.JPG "{{ site.baseurl }}/images/CredentialsStealer/12_Wndproc.JPG")

Taking a look inside this function:
![alt text]({{ site.baseurl }}/images/CredentialsStealer/13_WndprocIntrnl.JPG "{{ site.baseurl }}/images/CredentialsStealer/13_WndprocIntrnl.JPG")

We see that when this Window Procedure receives the `WM_CREATE` notification which is during the Window Creation, It calls the `SetClipboardViewer` API. According to MSDN: 
>SetClipboardViewer Adds the specified window to the chain of clipboard viewers. Clipboard viewer windows receive a WM_DRAWCLIPBOARD message whenever the content of the clipboard changes.

Therefore upon creation this window is inserted in the clipboard viewer chain.
For any subsequent calls to this Window procedure, if Msg is equal to 0x308 which is the value for the `WM_DRAWCLIPBOARD` constant as `#define`d in `WinUser.h`, which means whenever the content of the clipboard has changed according to MSDN above, it calls :
![alt text]({{ site.baseurl }}/images/CredentialsStealer/14_wmdrawclipboard.JPG "{{ site.baseurl }}/images/CredentialsStealer/14_wmdrawclipboard.JPG")

`OpenClipBoard` to open the clipboard relevant with the current task, and then `GetClipBoardData` to get a Handle to the Clipboard Data in `CF_TEXT` format. Then a string is initialised with the value \<WINDIR>\rundllx.sys.

![alt text]({{ site.baseurl }}/images/CredentialsStealer/15_clipstealer.JPG "{{ site.baseurl }}/images/CredentialsStealer/15_clipstealer.JPG")

The \<WINDIR>\rundllx.sys is passed to a function that acts as a wrapper over a `CreateFile` call, this creates a file rundllx.sys in the Windows Directory:
![alt text]({{ site.baseurl }}/images/CredentialsStealer/16_cliplogcreate.JPG "{{ site.baseurl }}/images/CredentialsStealer/16_cliplogcreate.JPG")

Then the handle to this File and the handle to the clipboard data in `CF_TEXT` format is passed to a function acting as a wrapper over a `WriteFile` call, which writes the clipboard data to the \<Windir>\rundllx.sys. So this window copies all clipboard data in `CF_TEXT` format and logs them to a file.
![alt text]({{ site.baseurl }}/images/CredentialsStealer/17_cliplogwrite.JPG "{{ site.baseurl }}/images/CredentialsStealer/17_cliplogwrite.JPG")

Next code of interest is:

![alt text]({{ site.baseurl }}/images/CredentialsStealer/18_sock64create.JPG "{{ site.baseurl }}/images/CredentialsStealer/18_sock64create.JPG")

Which creates a file WINDIR\sock64.dll and writes the second PE (hooks.dll) inside the
unpacked PE there. Then:
![alt text]({{ site.baseurl }}/images/CredentialsStealer/19_loadlibGetProc.JPG "{{ site.baseurl }}/images/CredentialsStealer/19_loadlibGetProc.JPG")

It loads the DLL into its Process Address Space via a call to `LoadLibrary` and receives the Handle to the DLL’s two exports `h_Init` and `h_Release` via calls to `GetProcAddress`. If we fire up x32dbg and look at this Loaded DLL in the process space:
![alt text]({{ site.baseurl }}/images/CredentialsStealer/20_dllimportresolved.JPG "{{ site.baseurl }}/images/CredentialsStealer/20_dllimportresolved.JPG")

We can see the export `h_Init`, `h_Release` have been decrypted and the imports have been resolved. At this point we can use savedata command in x32Dbg to dump this resolved DLL out and open it in IDA to see what the exports do:
![alt text]({{ site.baseurl }}/images/CredentialsStealer/21_hinitresolved.JPG "{{ site.baseurl }}/images/CredentialsStealer/21_hinitresolved.JPG")

As it’s shown here `h_Init` is just a wrapper around `SetWindowsHookExA` WinAPI which sets `sub_10001030` as a `WH_KEYBOARD` hook procedure for all the existing threads in the same desktop as the malware. According to MSDN, `WH_KEYBOARD` procedure is 'a hook procedure that monitors keystroke messages.' Looking at `sub_10001030` gives us .
![alt text]({{ site.baseurl }}/images/CredentialsStealer/22_hookfunc.JPG "{{ site.baseurl }}/images/CredentialsStealer/22_hookfunc.JPG")

It initialises a string “\<WINDIR>\bank.log” and passes it to a function `sub_1000121A`:
![alt text]({{ site.baseurl }}/images/CredentialsStealer/22_100121A.JPG "{{ site.baseurl }}/images/CredentialsStealer/22_100121A.JPG")

This subroutine takes us into a rabbit hole of obfuscation, however hunting down the routines a call to CreateFileA can be found in `sub_10004DFD` with this string as the first argument.
![alt text]({{ site.baseurl }}/images/CredentialsStealer/23_keylogcreate.JPG "{{ site.baseurl }}/images/CredentialsStealer/23_keylogcreate.JPG")

Then the Keycode supplied to this Hook procedure is converted to character via a call to `GetKeyNameTextA`:

![alt text]({{ site.baseurl }}/images/CredentialsStealer/24_GetKeyText.JPG "{{ site.baseurl }}/images/CredentialsStealer/24_GetKeyText.JPG")

Every keystroke by the user thus is converted to its corresponding character and logged in the \<WINDIR>\bank.log once the `h_Init` routine is called. 

Taking a look at h_Release we see it’s just a wrapper around UnhookWindowsHookEx which unhooks the procedure hooked by `h_Init`.
![alt text]({{ site.baseurl }}/images/CredentialsStealer/25_hrelease.JPG "{{ site.baseurl }}/images/CredentialsStealer/25_hrelease.JPG")

Therefore the two exports by this DLLs basically installs a keylogger hook function that logs all keystrokes by a user to a file, and the other export uninstalls this hook.

Going back to the malware, value of a Registry HKEY_LOCAL_MACHINE\Software\SARS is checked and in case it doesn’t exist infected machine’s IP is collected via calls to `gethostname` and `gethostbyname`, appended to some data in mail format and sent to smtp.mail.ru (the
malware’s C2C) which right now doesn’t have a valid DNS record.
![alt text]({{ site.baseurl }}/images/CredentialsStealer/26_localIPcollector.JPG "{{ site.baseurl }}/images/CredentialsStealer/26_localIPcollector.JPG")

![alt text]({{ site.baseurl }}/images/CredentialsStealer/27_smtp.JPG "{{ site.baseurl }}/images/CredentialsStealer/27_smtp.JPG")

Once this mail has been sent the the Registry HKEY_LOCAL_MACHINE\Software\SARS isadded and set the value to ‘start_bank’. So if the malware has run once in a machine this initial mail containing the local IP of the machine will be sent only once. This subroutine basically informs the Malware C2C of the IP it has successfully infected.

Now we enter the interesting part of the malware code. It initiates an infinite loop and first we see calls to:

![alt text]({{ site.baseurl }}/images/CredentialsStealer/28_barclaysIE.JPG "{{ site.baseurl }}/images/CredentialsStealer/28_barclaysIE.JPG")

`GetForegroundWindow` which retrieves handle to the topmost/foreground window and retrieves the Text at the foreground window title bar via a call to `GetWindowTextA`. Then it calls `FindWindowA` which returns a handle if the passed Classname and WindowName (foreground window Title text in this case) matches. The lpClassName passed to this call is “IEFrame” which is the predefined classname for Internet Explorer. So this `FindWindowA` call will only return a valid handle if the foreground window the user has open is a Internet Explorer Window. Then this handle is passed to a subroutine I labelled as `BarclaysIEChecker`. Peeking into this subroutine gives us:

![alt text]({{ site.baseurl }}/images/CredentialsStealer/29_findwindowchain1.JPG "{{ site.baseurl }}/images/CredentialsStealer/29_findwindowchain1.JPG")
![alt text]({{ site.baseurl }}/images/CredentialsStealer/30_Findwindowchain2.JPG "{{ site.baseurl }}/images/CredentialsStealer/30_Findwindowchain2.JPG")

It enumerates the Internet Explorer window open in the foreground by its child windows via WorkerA->rebarwindows32->comboboxex32->combobox->Edit, which returns a handle to the
address bar of the IE window. If a valid handle to the Address bar is retrieved:

![alt text]({{ site.baseurl }}/images/CredentialsStealer/31_sendmessageStrstr.JPG "{{ site.baseurl }}/images/CredentialsStealer/31_sendmessageStrstr.JPG")

A Thread Message to the caller/creator thread of the address bar window is sent with `Msg` parameter set to `WM_GETTEXT`. This will instruct the caller/creator thread of the address bar window to return the Text entered by the user in the address bar in the `lParam` pointer. This `lParam` string which now contains the domain name the user entered
into the Foreground IE window is passed to `strstr` function which searches a smaller substring in a larger string. The substring to be searched in the address bar text is
“ibank.barclays.co.uk/fp/” which is the index/login page of the barclays bank (although at the time of this write up this address is re-directed to a different domain address but it still contains the login functionality). If the substring is found which means the user has the Barclays bank login page open in their browser which is also currently  the foreground window so it is likely the user is entering their credentials into the page, the pointer to the address bar string is returned by the `BarclaysIEChecker` function. Then a call to `h_Release` is skipped if the Text inside the Address Bar contains the Barclays domain, which tells us the malware is interested in hooking and logging
the keystrokes of the user if they have the Barclays login page open on their browser.
![alt text]({{ site.baseurl }}/images/CredentialsStealer/32_hreleaseskip.JPG "{{ site.baseurl }}/images/CredentialsStealer/32_hreleaseskip.JPG")

Then the ForegroundWindow Title retrieved earlier is passed to a subroutine I labelled as `IWebBrowser2StorerGlobalCOMFlagSetter` (which basically describes all that it is doing).

![alt text]({{ site.baseurl }}/images/CredentialsStealer/33_COMstart.JPG "{{ site.baseurl }}/images/CredentialsStealer/33_COMstart.JPG")

This is where the Malware starts levearaging the COM framework exposed by Microsoft Windows. Therefore further discussion on this Malware warrants a second blog post
where the COM programming aspect of the Malware will be extensively discussed. So far we have seen the malware trying to make it persistent on the infected machine via several registry creation/copying itself to different executable files in the startup locations. It drops an ASPACK packed dll on the disk named \<WINDIR>\sock64.dll which is then `LoadLibrary`'d into the process address space during malware runtime. During loading the DLL exports are decrypted and it's imports are resolved. The DLL exposes two routines `h_Init` and `h_Release`. `h_Init` sets up a hooking procedure that logs all keypresses seen by every thread in the same desktop as the malware thread into a file, `h_Release` releases this hook. The Malware also creates a window that inserts itself into the clipboard viewer chain and logs all data user has copied to the clipboard to a file. Then the malware enters a loop, checks if the Foreground window the user has open is an Internet Explorer window or not, and in case it is if the user has the Barclay's bank login page open in it or not. If the user has the login page open, it skips a call the `h_Release` and continues further executing, which likely means it will call `h_Init` down the line and log all keypresses entered by the user in this Barclay's login page. Next the malware employs clever tricks to determine which stage of login the user is in and accordingly takes action to properly the log the keypresses, the nitti gritties of which will be discussed on part 2. Thanks for sticking around for so long with this write-up, stay tuned!

Contact: roysomik@yahoo.com / evilsapphire_s@yahoo.in
