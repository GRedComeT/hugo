---
title: 多线程基础
subtitle:
date: 2024-05-13T09:54:40+08:00
slug: 583bc6c
draft: false
author:
  name: Shiping Guo
  link: 
  email: spguo2000@gmail.com
  avatar:
description:
keywords:
license:
comment: false
weight: 0
tags:
  - Java
  - Concurrency
categories:
  - Java
hiddenFromHomePage: false
hiddenFromSearch: false
hiddenFromRss: false
hiddenFromRelated: false
summary:
resources:
  - name: featured-image
    src: featured-image.jpg
  - name: featured-image-preview
    src: featured-image-preview.jpg
toc: true
math: false
lightgallery: false
password:
message:
repost:
  enable: true
  url:

# See details front matter: https://fixit.lruihao.cn/documentation/content-management/introduction/#front-matter
---

# 多线程基础

## 并发与并行

### 并发执行

![bingfa](https://minio.dionysunrsshub.top:443/myimages/2024-img/bingfa.png)
同一时间只能处理一个任务，每个任务轮着做（时间片轮转），只要我们单次处理分配的时间足够的短，在宏观看来，就是三个任务在同时进行。
而我们Java中的线程，正是这种机制，当我们需要同时处理上百个上千个任务时，很明显CPU的数量是不可能赶得上我们的线程数的，所以说这时就要求我们的程序有良好的并发性能，来应对同一时间大量的任务处理

<!--more-->
