---
layout: post
title: "Punch hole"
date: 2014-09-30 15:23:46 +0800
comments: true
categories: 
---

We have some discussion about punch hole in a file before. Let me summary about this.

Sparse File
===========

We can create sparse file by "dd".

{% codeblock %}
dd if=/dev/zero of=test bs=1 count=0 skip=2G
{% endcodeblock %}
This cmd will create 2G file. But there isn't allocate any block for this file.
Then when you right date into the file, the file will become large.

fallocate
=========

We can allocate block for a file by command "fallocate".

{% codeblock %}
fallocate -l 100M test
{% endcodeblock %}

Then it will allocate 100M blocks for test file. But all those block is just allocated with
initilize, so it won't happend any I/O.

This is good for a file won't generate a lot of fragement.

Punch hole
==========

Punch hole can unallocated block in the middle of file. This can be used for image file of
virtual machine. If the some storage release, hypervisor can release those storage for
image file.

http://blog.csdn.net/g__gle/article/details/8083485

Also can reference: man 2 fallocate
