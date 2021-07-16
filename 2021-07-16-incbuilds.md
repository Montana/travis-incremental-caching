---
title: "Using cache and incremental builds in real world scenarios"
created_at: Fri Jul 16 2021 15:00:00 EDT
author: Montana Mendy
layout: post
permalink: 2021-07-16-incbuilds
category: news
excerpt_separator: <!-- more --> 
tags:
  - news
  - feature
  - infrastructure
  - community
---

![TCI-Graphics for AdsBlogs (1)](https://user-images.githubusercontent.com/20936398/125991579-9ef67346-70cc-461b-9f31-624c2d3c3276.png)

We all want shorter builds, what are some ways we can accomplish that? In this post I'll give you a variety of ways to help reduce build times, and overall optimization of your `.travis.yml` file. We'll talk about caching and the incremental build method in Travis CI. In general and the majority, short build durations are associated with Travis builds that are configured `cache` content that does not change often (e.g. bash scripts, some dependency), to finish as soon as all the
Travis `workers` finish the required jobs.


