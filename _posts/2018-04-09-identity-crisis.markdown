---
layout: post
title: Identity Crisis
date: 2018-09-04 21:29
published: true
categories:
---

I've long had a bit of a thing for 'classic' ThinkPads.  I owned a couple of X40s back in the day, and then a few years back I bought an X200 to lightly modify and run as an OpenBSD desktop.  There are lots to like about these machines - decent keyboard, reasonable build quality, and by virtue of their popularity within open-source circles, excellent hardware support for operating systems other than Windows (and macOS, obviously).  There's also a lot to hate about them, [especially under Lenovo's stewardship](https://www.techworm.net/2015/08/lenovo-pcs-and-laptops-seem-to-have-a-bios-level-backdoor.html), but for a certain vintage these problems can be disregarded or worked around.  

> _For a laugh and to break up this wall of text, here's the only photo I could find of one of my old X40s. It's a picture of my desktop at Carphone Warehouse, somewhere around 2004, with the ThinkPad off to the left looking classy in amongst all the corporate junk:_

![My desk at CPW](/public/static/cpw_x40.jpg)

However, if there's one thing that's always let them down it's the screens.  Every one I've had or used has been terrible.  The simple modification I did to the X200 was to upgrade its screen with an AFFS model but even so, it was still low resolution and was of very poor quality compared to my MacBook's.  I eventually sold it as it didn't really see all that much use, and since then I've not bothered thinking about them much.

My attention turned to them again recently though, prompted by a couple of things but mostly because my eye was caught a while ago by some really creative modders in China, with the technical chops required to transform models like the X61 and the X201 into machines [equipped with modern CPUs](https://liliputing.com/2018/03/x210-mod-turns-classic-lenovo-thinkpad-x201-into-a-modern-pc.html) and, most importantly (for me anyway), decent screens.

I'd be lying if I said it hasn't been motivated in part by Apple's attitude - perceived or otherwise - towards their 'traditional' (i.e non-iOS) devices.  It's something of a worry, and although I've not been affected by any of the issues with the hardware or software it's hard to ignore it completely. Maybe the new release of macOS and some refreshed non-Pro laptops, along with the re-designed Mac Pro will turn that around but it remains to be seen.

So anyway, I did my research, weighed up the pros and cons of [trying to do the mod myself](https://forum.thinkpads.com/viewtopic.php?t=122640), and eventually decided to wire nearly 400 UKP to a very random email address supposedly owned by one of the most respected modders (Jacky / [lcdfans](http://facebook.com/lcdfans)) in return for a relatively straightforward upgraded X230 with:

* An Intel i7-3520M CPU;
* A 2560x1440 12.5" screen;
* A motherboard modded to be able to drive the above display;
* A new shell;
* A new, modified 7-row keyboard from the previous (pre-chiclet) generation;
* Upgraded wireless (with support for 802.11ac);
* Upgraded Bluetooth (support for BT4.0).

Low and behold - and completely unannounced - it arrived just a few short weeks later.  I stuck 16GB in it along with a 1TB SSD and it's _awesome_.

Or rather, it was.  It worked for a day.  Then the display crapped out.  A few frantically-exchanged emails and videos later allayed any concerns I had that I'd be left high-and-dry with a non-working machine, as Jacky was extremely helpful and super responsive.  However, despite my best efforts at diagnosing the problem (under his direction) and an attempt at re-soldering the motherboard mod (!) I couldn't coax it back into life.  So I had to return it to China, which took a loooong time.

In fact the whole process took so long that the shine had almost completely worn off the whole idea.  The delay wasn't Jacky's fault, it was customs on either side taking their sweet time processing the shipment.  But I kept seeing other people on Twitter that had cottoned on to the availability of these machines as well and were showing off photos of them, working flawlessly, leaving me somewhat envious.  Finally though my replacement unit arrived and the great news is that it's worked magnificently ever since.

![The X230 in action](/public/static/x230.jpg)

As for as which OS I'm running... I had a brief dally with [NixOS](http://nixos.org), but there was a little too much friction (and potential fragility - mostly my fault) for what I need to do on a day-to-day basis. So instead I've settled on the latest release of [Fedora Linux](https://getfedora.org/) (I'm really bored of [orange](http://ubuntu.com/)) and - brace yourself - it's been wonderful.  I've been using it every day now for nearly a month and I've not - yet - switched back to my Mac full-time.

Everything works.  I get good battery life, about 7-8 hours for moderate usage from the new, official 9-cell battery I purchased.  I'm using [i3](http://i3wm.org) with [some bits](https://github.com/csxr/i3-gnome) to bring in the best Just Works components of GNOME and it's both fast and distraction-free, helping my concentration no end.  The screen is bright and clear, with decent viewing angles.  Unfortunately the resolution means that I have to run at 200% scaling for some bits of the UI; It'd be great if fractional scaling was properly supported in Linux so I could do 150% instead, but support for it just isn't quite there yet.  However, this isn't much of a problem in practice.

Here's a quick (staged) screenshot of i3 - bonus points if you recognise the background (the mouse mat I'm using in that previous photo provides a clue)...

![Screnshot of i3](/public/static/i3neofetch.png)

There's other benefits too.  The whole thing is properly _serviceable_.  If something breaks then there's a good chance I can just pick up a replacement part or hell, a machine entirely, on eBay for next to nothing.  The only precious bit is the modded motherboard, but odds are that it'll last (assuming I've not just jinxed it by saying that).  Also it's refreshing not to have to baby the thing to quite the same degree with a MacBook.  If I drop it and crack the case then no problem, it's pennies to replace and you can DIY.

Saying all that though, I can't hand-on-heart say goodbye to Apple's macOS entirely.  Despite the recent mis-steps, I still think that macOS is the best desktop operating system and that Apple make the best laptop hardware.  But it's _fun_ using Linux on the desktop again for the first time in a while, and for the job I'm doing right now it's actually more suitable than macOS is.  That's a topic for another post though...

If you're interested in one of these machines (and you should be, especially given the price and limited availability) then Jacky has a new website [here](http://cnmod.cn) and a Facebook page [here](https://facebook.com/lcdfans).  Tell him I said "hi" - we exchanged about 80 emails during the course of this whole process so I'm sure he misses me 😉
