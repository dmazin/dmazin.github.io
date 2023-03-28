---
layout: post
title: "Let’s play: I've lost my website!"
byline: By <a href="http://cyberdemon.org/">Dmitry Mazin</a>.
date: 2021-05-19
---
Remember [this Bash quote](http://www.bash.org/?5273)?

```
<erno> hm. I’ve lost a machine.. literally _lost_.
it responds to ping, it works completely,
I just can’t figure out where in my apartment it is.
```

Well, I’ve lost my website (the one you're reading right now). It loads, it works completely, I just can’t figure out where I’m hosting it, what the source repo is, or how I built it in the first place.

So… let’s find it?
First, where does the domain name even point?

```
~ dig archvile.net

; <<>> DiG 9.10.6 <<>> archvile.net
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 45923
;; flags: qr rd ra; QUERY: 1, ANSWER: 4, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;archvile.net.                  IN      A

;; ANSWER SECTION:
archvile.net.           2363    IN      A       185.199.111.153
archvile.net.           2363    IN      A       185.199.110.153
archvile.net.           2363    IN      A       185.199.109.153
archvile.net.           2363    IN      A       185.199.108.153
```

Alrighty, what are these 185.199.*.153 IPs?

```
~ curl 185.199.111.153 -I
HTTP/1.1 404 Not Found
Server: GitHub.com
```

Ah, GitHub. Is it a GitHub page?
Right, I do have a [GitHub page repo](https://github.com/dmazin/dmazin.github.io) with awfully similar content to the site. But is this **the** repo?

Let me just change some text and see if the change gets reflected.
Let’s add `<meta name=‘poo’ content=‘dumb’>` to index.html:

```
From 9b92abcfae3185f9140b20414da67cad19b0b272 Mon Sep 17 00:00:00 2001
From: Dmitry Mazin <363966+dmazin@users.noreply.github.com>
Date: Wed, 19 May 2021 13:05:45 -0400
Subject: [PATCH] Update index.html

---
 index.html | 1 +
 1 file changed, 1 insertion(+)

diff —git a/index.html b/index.html
index 564ed04..e0ed398 100644
— a/index.html
+++ b/index.html
@@ -14,6 +14,7 @@
 </script>
 
   <meta name=‘viewport’ content=‘width=device-width’>
+  <meta name=‘poo’ content=‘dumb’>
   <link rel=‘stylesheet’ href=‘/assets/index.css’>
 </head>
```

(You can generate such a patch by appending `.patch` to any GitHub commit URL, like [https://github.com/dmazin/dmazin.github.io/commit/9b92abcfae3185f9140b20414da67cad19b0b272.patch](https://github.com/dmazin/dmazin.github.io/commit/9b92abcfae3185f9140b20414da67cad19b0b272.patch)).

And so… there it is! (The `-s` suppresses curl’s progress bar.)
```
~ curl -s https://archvile.net | grep poo
  <meta name='poo' content='dumb'>
```

Now, where do I manage the DNS settings for archvile.net? I actually remember that I used Google Domains.  To confirm:

```
~ whois archvile.net | grep "Registrar URL"
   Registrar URL: http://domains.google.com
```

Mystery solved!

I’ll leave the poo meta tag as a present for the future.
