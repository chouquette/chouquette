---
layout: post
title: "VideoLAN and HTTPS"
tags:
- videolan
---

A few days ago, the good old "VideoLAN must use HTTPS" topic came back, in the form of a [bug report](https://trac.videolan.org/vlc/ticket/18472). However this time, 
discussion heated up way above proportions on twitter, so I'd like to take a step back and
address what's the status (or rather, my own view on the status) are.

Speaking of views, this post is about my view and understanding of the topic, and mine only. This is not a VideoLAN statement,
this is just how I understand things, and why I believe the answer to the everlasting question
"Should VideoLAN enforce HTTPS for all there downloads" is the most common answer in IT: "It depends."

The base statement is "if you don't enable HTTPS anyone can Man In The Middle attack your connection and replace your VLC install with a malware". The first part is true, the 2nd is much more complicated.

I kind of feel obligated to comment on the bug report part about the lack of HTTPS being how the [CIA used VLC to take control of computers](https://wikileaks.org/ciav7p1/cms/files/Rain%20Maker%20v1.0%20User%20Guide.doc)
. The answer here is much simpler: this is false.

They used the application manifest to force a malicious DLL to be loaded instead of the system provided ones. The VLC installation was provided on a USB key, and to be honest this is
not much more different from simply recompiling VLC with whatever extra code you want, since you rely on the victim trusting you and wilfully executing the binaries your provided them.

In short, we mitigated this attack vector by:
* Embedding the manifest in the application
* Refusing to load any DLLs from the application directory

The first part is interesting, because now the manifest is part of the final executable, meaning one needs to modify the the binary to change it, and this is important, as you'll below.

But let's back on the HTTPS topic

It is indeed true that using HTTP is insecure. I don't think anyone argues on that part. You're sending stuff for the whole world to see, and potentially intercept and edit.

It is also true that our frontpage doesn't enforce HTTPS by default, and that Google doesn't redirect you to the HTTPS page. We are investigating the later, the former is complicated due to old
browsers, which we still have to support (you'd be surprise how many Windows XP users we still have)

However claiming that videolan.org doesn't support HTTPS would be [wrong](https://www.videolan.org)

![Green Locker!](/img/posts/https/green-locker.png)

In anycase, there is no arguing that enabling HTTPS everywhere would be a good thing. The sad part is, we can't.

VideoLAN relies on [mirrors](https://get.videolan.org/vlc/2.2.6/win32/vlc-2.2.6-win32.exe?mirrorlist)
to allow us to handle the [bandwidth required](https://get.videolan.org/?mirrorstats) to distribute VLC. There is no way we could do that with our own infrastructure, based on donations only.
Those mirrors are gracefully provided to us for free by some companies (and we are very grateful to them!), but we do not own the machines, nor are we in a position to make any kind of demands to them.

Some people suggested that we use Cloudfare, but that is not an option, as they are US based, and must therefore comply with US laws, which has a different view on software pattents
than Europe. If we were to switch to Cloudfare, and they were required to stop hosting us, we'd have no way to provide VLC to our users, and no mirror infrastructure left. This is not an option.
Oh and the hypothesis of having a mirror shutting down because of legal heat is not an hypothesis, it already happened.

But ok let's say we have all our mirrors switching to HTTPS. Actually quite a few of them already are (and we are, again, very grateful to them!), but we don't control what gets out of the mirror.

Our mirror system ([mirrorbits](https://github.com/etix/mirrorbits)) already take quite a few actions to try and ensure the proper files are being served:
* Every minute, a random file is checked for integrity and availability
* Files synchronization twice per hour, on each mirror
* enforcing HTTPS for mirrors that support it

If we detect something dodgy, we will remove the mirror from our ring for some time, or potentially forever.

But a rogue mirror could still decide to serve you the file it wants while reporting something different to mirrorbits (spoiler alert: it happened). So even with an HTTPS mirror
you can get a malicious file (but hey! It has a green locker!). This is not something mirrorbits can ensure, and not what it was made for.
This is why we chose to focus our security measures on the binaries being downloaded.

All our binaries are signed with both GPG, and our Authenticode signature on Windows, our Gatekeeper signature on macOS.

For those who are not familiar with Windows/macOS, Authenticode and Gatekeeper are *basically* the operating system provided way of checking the authenticity & sanity of a binary.

From there, multiple configurations are possible:

On First install: it is the operating system's responsability to ensure the binary is signed, and matches our signature.

If the Authenticode signature matches, then all is good! You can see that it is being distributed by VideoLAN, but more importantly, no warning, nothing worrying.
![Genuine VLC installer](/img/posts/https/vlc-normal.png)

However, if the binary has been tampered with, you can see & feel the warning instantly. So if you were to modify a few bytes of, let's say the application manifest, you'd get
a nice warning telling you this is unsafe.
![Tampered VLC installer](/img/posts/https/vlc-tampered.png)

And if you're running an unsigned binary, you have to actively want to run it, as Windows won't let you do it by default
![Unsigned VLC installer](/img/posts/https/vlc-unsigned.png)

This leaves us with those possibilities:

* The MITM attack replaces the entire binary with another unsigned malicious binary. In this case, Windows should tell the user to back off. It is still possible to install it, but you are
basically warned that you are on your own.
* The MITM attack tampers the binary, which leads to quite visible windows warning.
* The MITM attack replaces the installer by an older VLC version (signed & untampered, but with known flaws). This is the most worrying one in my opinion. This is only valid at first install,
as the installer will warn the user about a downgrade if VLC is already installed. That being said, at first launch, VLC will check for updates.
* The MITM attack replaces the installer with an authenticode signed malware. This is theoretically possible, but if you don't trust certification authorities, I don't think you can trust HTTPS.

You could argue (and I believe you'd be right) that the signed but tampered warning could be more extreme, but we sadly can't control that.

I don't have a macOS computer at hand, but I believe the output to be quite similar.

When it comes to the updated, it will check for the GPG signature (the public key is embedded in the VLC install). The updater will refuse anything that is:
* Downgrading VLC
* Not matching the GPG signature.

That makes an HTTP download completely safe as far as I can see.

So the only pertinent attack vector would require the user not to know which version they install (that is not unlikely), but also to refuse update checks, or the update offer that will pop on first launch if they do accept the check.

I will emphasize that we do *not* require the user to explicitely check the GPG signature nor the Authenticode/Gatekeeper signature. The updater/installer will do it for them.

However as described earlier, there is still a small attack vector left, between the first install and the second launch, and we need to address this at some point.
Now keep in mind that due to the mirrors not being ours, we still can't enforce HTTPS. However we could rely on another mechanism. I strongly encourage you to vote for this (very old)
suggestion on Mozilla's bugtracker: 

[https://bugzilla.mozilla.org/show_bug.cgi?id=292481](https://bugzilla.mozilla.org/show_bug.cgi?id=292481)

This would allow us to provide our checksums (which is ultimately the best way of ensuring that a user downloads the correct version, HTTP or HTTPS, provided that the checksums gets
distributed over a secure connection). We can't expect a user to know about checksums (some do, and that is great, but most don't). We can, however, expect software they use to do so.

There is also an obvious thing we can all do, and that is to educate users about the importance of updating. We also have to do our share to ensure there aren't too drastic regressions from a version to another.

I believe VideoLAN is working hard on those matter.

Hopefully that sheds a bit more light on the situation, and will help move away from the "No HTTPS means total unsecurity" point of view. I agree that in a perfect world, 
we'd deploy our own servers worldwide, would enable HTTPS for all, but we can't.

Someone said today that "Google moved to HTTPS only at no impact to their capacity". I'll glady point you to the fact that Google is not a non-profit, and that the amount of money they make
in a day is an order of magnitude (hello euphemism!) higher than the amount of money we ever received from donations. Ever.

A few closing remarks, in no particular order, from what I read today:
* I don't claim to know better, this is my understanding (which I believe to be accurate), but I am not a security specialist, and I don't claim to be better than anyone. If you
feel that I missed something or want to discuss it, I'll gladly engage. However, if the summary would be "You don't know shit, let me explain", well, get lost.
* Claiming that we are wilfully allowing our binaries to be tampered with is outrageously wrong, and actually quite hurtful. We refused countless offers to monetize VLC because we believe
that when you download VLC, you only want a media player, with no strings attached, no tracking, no add, and no malware. I've spent enough time removing crap from my relatives' computer
not to do this to anyone.
* Implying that we don't care about opressive regimes spying on their citizens in order to torture and oppress them is low, hurtful, false, and quite disgusting of an argument to make.
* If you use homosexuality in order to insult/make fun of someone, it is homophobic. You might disagree, but that's irrelevent. It is.
* Just be nice to each other, let's learn from each other, and let's make the web & software safer. Yes that is cheesy, but if that isn't what you want, you probably don't belong around open source community.

