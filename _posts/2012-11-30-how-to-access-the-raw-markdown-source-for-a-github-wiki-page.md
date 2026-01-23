---
layout: post
title: "How to access the raw markdown source for a github wiki page"
date: 2012-11-30 01:44:00 +0000
tags: [ddd, ef, eventsourcing, extensibility, gadgets, mocking, moq, msbuild, mvc, nuget, patterns, programming, t4, technology, vsx, wcf, webapi]
---

##  [How to access the raw markdown source for a github wiki page](http://blogs.clariusconsulting.net/kzu/how-to-access-the-raw-markdown-source-for-a-github-wiki-page/ "How to access the raw markdown source for a github wiki page")

November 30, 2012 1:44 am

This is not entirely obvious (at least it wasn’t for me), but since Github wikis are actually backed by a [proper Git repo](https://github.com/blog/699-making-github-more-open-git-backed-wikis), I figured it should be possible to access the raw markdown for a page by using Github’s <https://raw.github.com/> style URLs.

After some minor trial/error, it turns out to be very predictable (as many things in github):

https://raw.github.com/wiki/[user]/[project]/[page].md

Just replace the square brackets’ stuff with your values and that’s it. I think I’ll be trying this out with a wiki page as a Release Notes landing page, which I just pull in raw format on a build script and replace the Nuget <releaseNotes> node with its content…

/kzu
