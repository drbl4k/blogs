+++
title = "In the Plain Sight (A Classic Filter Mistake)"
date = "2025-07-28T01:35:14+05:30"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
author = "drbl4k"
authorTwitter = "" #do not include @
cover = ""
tags = ["Local File inclusion", "SSRF", "Burpsuite"]
keywords = ["Burpsuite", ""]
description = "A default proxy filter mistake"
showFullContent = false
readingTime = true
hideComments = false
+++

> ***Prologue***: Being in application security, I usually see mates making this one tiny mistake with their Burpsuite configuration. Lot of testers usually go with the default  “HTTP History Filters”. In this blog I will cover the importance of the unchecked filters and how it helped me as well as fellow hackers out in the wild to attain **“Path Traversal”** and **“Server-Side Request Forgery”** in a real engagement.
> 

---

Most of public reports (*around 55%*) in **Hackerone** indicate that, the components that are vulnerable to local file inclusions and SSRFs are identified to be either a functionality of an image/other files `(*ie: PDF, CSV, ZIP etc.*)` being loaded into the application through an external interaction `*(ie: ?image=https://test.com/a.pdf)`* or fetched locally within the file system of the hosted application `*(ie: ?filename=test.png)*`. **

![image.png](/images/in-the-plain-sight/image.png)

*Source: [https://vickieli.dev/ssrf/ssrf-in-the-wild/](https://vickieli.dev/ssrf/ssrf-in-the-wild/)*

It’s fascinating, because, we as Pentesters usually go after the core functionalities of an application, that could or couldn’t be vulnerable to such attacks, are really missing this perspective of attack vector all together. Let’s leave application pentest realm for a second, imagine conducting an external pentest, and encountering an application vulnerable to a full read SSRF attack and letting it slide because of the default filters. 

---

I would like to share my experience on one of my recent pentest activity, where almost on the last day I got hold of a path traversal vulnerability which led to disclosure of internal files. 

Real Application Details:

- Microsoft IIS server v10
- Internal web app (can be accessed within a restricted environment)
- Has multiple features to download CSV and ZIP files.

---

On the last day I usually go for analysis of images and other files like PDFs and Excel for metadata exposure and upon applying the Burpsuite filter, one endpoint stood out. It was an endpoint trying to fetch a .zip file from the internal file system.  Now obviously it won’t be visible in the HTTP history due to receiving a binary file **(*garbage data*)** as response. The request had an interesting parameter which was pulling a zip file `/download/zip?fnames=UserLog\admin.zip&sec=zip`. **It looked something like this in the simulated environment.

![4.png](/images/in-the-plain-sight/4.png)

We all know what would have been my next move. Yes I tampered the `?fnames=` parameter with the payload `..\..\..\..\..\testfile` (Note, since the real request had a backslash with the file name, I used a traversal payload with backslashes). The real response for the request was a stack trace error indicating that `"The file was not found in the path E:\testfile"`. **(*Pro Tip: something as simple as a stack trace could reveal a lot about the backend.*)** Bingo! as soon I got the error indicating the internal file path I figured out where I was, and, I tried to fetch the “web.config” file **(*Its a default multi purpose configuration file for IIS servers*)** from the “E:” directory with the following payload `..\..\..\..\..\web.config`. It looked something like this.

![5.png](/images/in-the-plain-sight/5.png)

![6.png](/images/in-the-plain-sight/6.png)

The web.config file usually throws an error of some kind (403 or 404 usually) if we try to access it directly through a web browser, but because of this path traversal vulnerability, I was able to access it and show the impact. In my case the file consisted details about the SMTP configuration with keys present in it. (*I was on top of the world*)

---

Moral of the story, never trust any (*and I mean **ANY***) of the functionalities. By assuming so, we may giveaway the chance to find something cool. Even something as simple as a **default Burpsuite filter**, which hides image and binary data will act against us. Let’s be honest, who doesn’t want to find and exploit a local file inclusion or an SSRF? not me at least.

---

References:

- https://hackerone.com/reports/1888808
- https://hackerone.com/reports/1427086
- https://vickieli.dev/ssrf/ssrf-in-the-wild/

---

**Happy Hunting !**
