---
layout: post
title:  "Use Alpine-based Docker images to reduce the build time and disk space usage"
date:   2021-04-18 18:36:00 +0100
categories: docker alpine build
---

![Alpine Linux Logo](https://alpinelinux.org/alpinelinux-logo.svg "Alpine Linux Logo")

Many projects use Docker images for development and production purposes. The images are used as-is
or used to create new ones.

In both cases the build time can be saved just by adding "-alpine" to the version, you reduce the
download time so.

![The size of non-Alpine-based and Alpine-based images](/assets/2021-04-18-alpine-based-docker-images.png "The size of non-Alpine-based and Alpine-based images")

As a bonus, you save the disk space needed to store the image locally.
