---
layout: post
title: Android中的系统属性
categories: property
tags: property
published: false
comments: true
---

* content
{:toc}

Android系统属性是以键值对<Name,Value>的形式体现的。在整个系统中国年是一个全局比变量，进程可以通过get/set取操作属性的值。一旦系统启动，Android会分配一块共享内存来存储这些属性，这个工作是在init进程中完成的。init会启动PropertyService，每个性药访问系统属性的客户进程都属性要通过PropertyService去访问。
