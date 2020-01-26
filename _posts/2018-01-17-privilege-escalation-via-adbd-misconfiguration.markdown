---
layout: post
title:  "CVE-2017-13212 - Privilege Escalation via adbd Misconfiguration"
date:   2018-01-17 07:26:16 +0000
categories: jekyll update
---

I've released an advisory which details and demonstrates a vulnerability that exists in Android's 'ADB over Wifi' feature. This vulnerability when exploited would allow any android application to elevate its privileges from the context of “u:r:untrusted_app:s0” to that of “u:r:shell:s0”.

This vulnerability currently affects Android 4.2.2 to 8.0. The patch for which was included in the January 5, 2018 Android Security Bulletin. Android users are advised to update their devices to this patch level.

The advisory is hosted at [MWR Labs][mwr-labs]

[mwr-labs]:https://labs.mwrinfosecurity.com/advisories/privilege-escalation-via-adbd-misconfiguration/