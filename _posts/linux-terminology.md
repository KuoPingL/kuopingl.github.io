---
layout: post
title:  "Linux - Filesystem"
date:   2022-08-08 16:15:07 +0800
categories: [linux, beginner]
---



### Filesystem
In Linux, there are

**What is a file descriptor ?**
> A file descriptor is an unsigned integer used by a process to identify an open file.
> <br><br>The number of file descriptors available to a process is limited by the /OPEN_MAX control in the sys/limits. h file. The number of file descriptors is also controlled by the ulimit -n flag. [source][1]
> <br><br>It describes a data resource, and how that resource may be accessed. [source][2]

When a program asks to open a file or other data source, the kernel will perform actions similar to network socket [2][2]:
- Grants access
- Creates the entry in **global file table**
- Provides the software with the location of that entry.



[1]: https://www.ibm.com/docs/ssw_aix_71/com.ibm.aix.genprogc/using_file_descriptors.htm
[2]: https://www.computerhope.com/jargon/f/file-descriptor.htm
