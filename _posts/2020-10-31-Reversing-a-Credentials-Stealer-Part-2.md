---
layout: post
title: Reversing a Barclays Bank Credentials Stealer (Part 2)
---

Welcome to part 2 of the analysis work of a Barclays Bank credentials stealer. Last time we left the discussion at the complicatedly named `IWebBrowser2StorerGlobalCOMFlagSetter` Function which despite my convoluted naming scheme simply describes all that it is doing.
![alt text]({{ site.baseurl }}/images/CredentialsStealer/33_COMstart.JPG "{{ site.baseurl }}/images/CredentialsStealer/33_COMstart.JPG")

This function is where the Malware starts leveraging the COM framework provided by Microsoft. At the very start we see a call to `CoCreateInstance` which creates an interface with a specified RIID of a COM object associated with the specified CLSID.

![alt text]({{ site.baseurl }}/images/CredentialsStealer2/1_cocreateinstance.JPG "{{ site.baseurl }}/images/CredentialsStealer2/1_cocreateinstance.JPG")



