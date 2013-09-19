---
layout: post
title: "How to mount a SMB share on Mac OS X"
date: 2013-09-19 13:17
comments: false
categories: 
---

Mounting a remote SMB share on Mac OS X is done using the `mount_smbfs` tool. Here's how to do it:

```sh
#Create the mount point:
mkdir share_name

#Mount the share:
mount_smbfs //username:password@server.name/share_name share_name/
```
When you're done with the share, unmount it with the following command:

```sh
umount share_name/
```
Hope that helps!

