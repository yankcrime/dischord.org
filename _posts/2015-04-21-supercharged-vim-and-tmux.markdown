---
layout: post
title: Supercharged terminal usage, starring zsh, vim and tmux
date: 2015-04-21 12:18
published: true
categories:
---

This is one of those things that I've been meaning to document for years now but have never gotten around to for a few reasons, chiefly amongst which is the fact that my setup is almost constantly evolving. I learn a lot from looking at other people's setups, screenshots, videos, and so forth but until now haven't bothered to document my own configuration. Generally speaking though the basics and the core functionality haven't really changed in a long timeso it's about time I made some notes on what I've set up and how.  A lot of this post is specific to use under OS X, but there's a few bits here and there that apply cross-platform.

Here's what this setup will look like once installed:

![vim in action](http://dischord.storage.datacentred.io/static/vimshot1.png)

The devil's in the details though, i.e in the workflow itself.  For example, this is a split-pane vim session inside a split-pane tmux session, and I'm navigated between the two with the same set of keybindings.  How?  Magic.

# The basics
Like most things vim-related, it all starts with Tim Pope.  Well, his plugins anyway. [vim-pathogen](https://github.com/tpope/vim-pathogen) is pretty much the first thing you want to install, as without this you're in for a world of pain when it comes to installing any other vim plugin.  That might be a slight exaggeration but trust me when I say that it makes life a lot, lot easier.  In conjunction with `vim-pathogen`, it might be that you only need one other addition to the default vim setup - [vim-sensible](https://github.com/tpope/vim-sensible).  As its name suggests it's a sensible set of defaults that almost everyone can agree on, regardless of which editor they've used previously.  If I need to use vim remotely for basic tasks then generally this is the only plugin I need to be productive and manipulate text via vim in a familiar, comfortable way.

For everything else, read on.

# The terminal
I use iTerm2 for one simple reason - the non-native fullscreen mode that it supports.  I don't particularly like the UI delay that OS X's native fullscreen / workspace mode introduces, and iTerm2 has support for a fullscreen mode that stays on the single workspace and leaves just the menubar exposed.  There's an awful lot of additional functionality that this terminal client introduces and to be honest I don't really most of it, save for its enhanced support for colour profiles and being able to easily import one of the base16 colour schemes.

My default shell is zsh, having switched away from bash a while ago for interactive usage after discovering a couple of zsh plugins that make using the commandline even more vi-like.  The standout plugin in particular is opp.zsh, which gives you vim-like text objects right within your shell, meaning you can express things like `"cw"` to change a word, `"ci"` to change all characters within a pair of speechmarks, and so on.  Otherwise I don't bother with the really bloated (in my opinion) configuration 'frameworks' like oh-mh-zsh or prezto, instead preferring to keep my .zshrc fairly lean and only containing settings that I understand and have placed there personally.

