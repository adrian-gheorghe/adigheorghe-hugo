---
title: How to host your static website on Github Pages
excerpt: Lately I’ve noticed I’ve been paying quite a bit of money on hosting. Well not really a lot, but I thought I could do with optimising what…
date: 2019-04-23
hero: "/img/1__LFiJenS3MWyUQ6zElDdv9Q.jpeg"
timeToRead: 2
authors:
  - Adrian Gheorghe
---

[Published on Medium](https://medium.com/@adrian.gheorghe.dev/how-to-host-your-static-website-on-github-pages-a3e1309bb91) [Image Credits](https://www.pexels.com/photo/grayscale-photo-of-computer-laptop-near-white-notebook-and-ceramic-mug-on-table-169573/)

Lately I’ve noticed I’ve been paying quite a bit of money on hosting. Well not really a lot, but I thought I could do with optimising what servers I was using and how. So I thought I could start by moving some projects to free services. The first one that came to mind was my static website/ online cv. This I thought would make a great candidate for Github pages hosting.

The process was a lot easier than I thought, and even if it doesn’t mean much, if I manage to move another 2 small WordPress sites, I will be able to discontinue a server I’d been paying for, which is something. There are tons of other posts on this and I’m about 3 years late to the party, but here are the steps I followed:

*   Move the code to a Github repository named your **userhandle.github.io**
*   Add **CNAME** file to your repo containing your custom domain
*   Add the following **A record** ips to your DNS

```bash
185.199.111.153
185.199.110.153
185.199.109.153
185.199.108.153
```
*   Add the following CNAME dns record

```bash
http://www.domain.org — userhandle.github.io.
```

*   Enable https from your repository settings, under Github Pages, Enforce HTTPS.

Extra information available here: [https://pages.github.com/](https://pages.github.com/)

