---
layout: post
title: Write-up on Reversing a Credentials Stealer (Part 1)
---

I recently had my hands on a malware sample that came with a malicious e-mail attachment. Upon reversing it I found out it employed various facets of Windows Programming to steal Barclay's Bank credentials from an unsuspecting user. It took advantage of Windows GUI programming flexibility to insert a Window inside the clipboard chain to log user clipboard data. It also extensively used the COM framework to determine the user behaviour and accordingly steal user’s data. The malware definitely warrants a entry in this personal Malware Analysis write-up blog.

The file that I renamed as 'mal.fil' is a simple PE file that can be opened with a PE viewer tool like PE Bear.
![alt text]({{ site.baseurl }}/images/CredentialsStealer/1_file.JPG "{{ site.baseurl }}/images/CredentialsStealer/1_file.JPG")

If we take a look at the sections, we can clearly see this is a file packed with UPX packers as indicated by the sections names UPX0, UPX1..
![alt text]({{ site.baseurl }}/images/CredentialsStealer/2_UPXsections.JPG "{{ site.baseurl }}/images/CredentialsStealer/2_UPXsections.JPG") 
