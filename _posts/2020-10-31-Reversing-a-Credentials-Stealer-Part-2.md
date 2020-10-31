---
layout: post
title: Reversing a Barclays Bank Credentials Stealer (Part 2)
---

Welcome to part 2 of the analysis work of a Barclays Bank credentials stealer. Last time we left the discussion at the complicatedly named `IWebBrowser2StorerGlobalCOMFlagSetter` Function which despite my convoluted naming scheme simply describes all that it is doing.
![alt text]({{ site.baseurl }}/images/CredentialsStealer/33_COMstart.JPG "{{ site.baseurl }}/images/CredentialsStealer/33_COMstart.JPG")

This function is where the Malware starts leveraging the COM framework provided by Microsoft. At the very start we see a call to `CoCreateInstance` which creates an interface with a specified RIID of a COM object associated with the specified CLSID.

![alt text]({{ site.baseurl }}/images/CredentialsStealer2/1_cocreateinstance.JPG "{{ site.baseurl }}/images/CredentialsStealer2/1_cocreateinstance.JPG")

The supplied `CLSID` and `RIID` are stored variables in this binary, taking a look at them in IDA Hex view gives us,
`CLSID` in little endian format:
![alt text]({{ site.baseurl }}/images/CredentialsStealer2/2_clsidhexview.JPG "{{ site.baseurl }}/images/CredentialsStealer2/2_clsidhexview.JPG")

`CLSID` is a globally unique identifier of a COM Object which can be found in the Registry:

![alt text]({{ site.baseurl }}/images/CredentialsStealer2/3_regclsid.JPG "{{ site.baseurl }}/images/CredentialsStealer2/3_regclsid.JPG")

Therefore the COM Object associated with the CLSID is ShellWindows which contains a collection of all Explorer/Internet Explorer Windows User has opened. Taking a look at the `RIID` we see in IDA Hex View:

![alt text]({{ site.baseurl }}/images/CredentialsStealer2/4_riidhexview.JPG "{{ site.baseurl }}/images/CredentialsStealer2/4_riidhexview.JPG")

In the ExDisp.h header file shipped with the Windows SDK we find the reference to this `RIID`:

![alt text]({{ site.baseurl }}/images/CredentialsStealer2/5_exdispriid.JPG "{ site.baseurl }}/images/CredentialsStealer2/5_exdispriid.JPG")

Therefore this call is asking a `IShellWindows` Interface to the `ShellWindows` COM Object. Then the Method at the 0x1C offset in this interface is called:

![alt text]({{ site.baseurl }}/images/CredentialsStealer2/6_ishellwindowsvtable.JPG "{{ site.baseurl }}/images/CredentialsStealer2/6_ishellwindowsvtable.JPG")

In the same ExDisp.h header we find the struct definition of the IShellWindows Virtual table. Counting the offset we find the method to be `get_count`:

```
typedef struct IShellWindowsVtbl
{
BEGIN_INTERFACE
HRESULT ( STDMETHODCALLTYPE *QueryInterface )(
__RPC__in IShellWindows * This,
/* [in] */ __RPC__in REFIID riid,
/* [annotation][iid_is][out] */
_COM_Outptr_ void **ppvObject);

…

HRESULT ( STDMETHODCALLTYPE *get_Count )(
__RPC__in IShellWindows * This,
/* [retval][out] */ __RPC__out long *Count);

…

END_INTERFACE
}
```

`IShellWindowsVtbl.get_count` returns the count of all open Internet Explorer Objects in the count Variable on the stack. This count variable is then used as a counter in a loop to iterate over all the open Internet Explorer Objects in the infected machine:

![alt text]({{ site.baseurl }}/images/CredentialsStealer2/7_ieobjectiteration.JPG "{{ site.baseurl }}/images/CredentialsStealer2/7_ieobjectiteration.JPG")

For each Internet Explorer object the `IShellWindowsVtbl.Item()` method is called which returns the IDispatch Interface for the registered shell window in the `IDispatch` variable labelled on the stack.
Then the first Method in the IDispatch VTable is called:

![alt text]({{ site.baseurl }}/images/CredentialsStealer2/8_queryinterfacecall.JPG "{{ site.baseurl }}/images/CredentialsStealer2/8_queryinterfacecall.JPG")

From OAIdl.h supplied with Windows SDK, this method turns out to be `IDispatchVtbl.QueryInterface()`:

![alt text]({{ site.baseurl }}/images/CredentialsStealer2/9_idispatchvttable.JPG "{{ site.baseurl }}/images/CredentialsStealer2/9_idispatchvttable.JPG")

`QueryInterface()` returns an Instance in the ppvObject associated with the supplied RIID. In the call the supplied RIID is yet another Global Variable at address 0x437B28 which can be viewed in the IDA Hex Viewer to be:

![alt text]({{ site.baseurl }}/images/CredentialsStealer2/10_iwebbrowserriid.JPG "{{ site.baseurl }}/images/CredentialsStealer2/10_iwebbrowserriid.JPG")

From ExDisp.h again it can be found that the corresponding interface is an `IWebBrowser2` Interface:

![alt text]({{ site.baseurl }}/images/CredentialsStealer2/11_iwebbrowserheaderfile2.JPG "{{ site.baseurl }}/images/CredentialsStealer2/11_iwebbrowserheaderfile.JPG")


(Entry in progress)



