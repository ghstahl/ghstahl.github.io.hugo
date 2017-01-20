+++
title = "CDN Origin Server Redirector Technique"
draft = true
date = "2017-01-21T08:51:52-08:00"
tags = ["ASP.NET Core","CDN"]
categories = ["ASP.NET Core"]
slug = ""

+++

## Problem
I had the joy of setting up a CDN configuration in Akamai.  Though it wasn't that bad, the rarity of that task made me never want to do it twice.
I only really needed to CDN cache static assets, and chose to harden the rules for anyone that wanted to use my configuration.

## The Rules.
1. I am going to set the CDN time to live for 1 year
2. I will NOT be clearing your CDN cache just because you made a mistake

