---
layout: post
title: Write-up on Reversing a Credentials Stealer (Part 1)
---

I recently had my hands on a malware sample that came with a malicious e-mail attachment. Upon reversing it I found out it employed various facets of Windows Programming 
to steal Barclay's Bank credentials from an unsuspecting user. It took advantage of Windows GUI programming flexibility to insert a Window inside the
clipboard chain to log user clipboard data. It also extensively used the COM framework to determine the user behaviour and accordingly steal user’s data. The malware definitely warrants 
a entry in this personal Malware Analysis write-up blog.

