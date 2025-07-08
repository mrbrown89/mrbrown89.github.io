---
title: "Salting GitOps. Managing a NAS with Salt and GitOps"
date: 2025-07-06
categories: [linux]
tags: [gitops, storage, salt]
layout: single
---

We have two NAS systems. One of which is our backup NAS which was built in house by me. This NAS has a few functions. It contains a copy of all data which is continually being synced to it, performs filesystem snap shots on the data, provides access to the data when the main NAS fails and runs our backup software.

This NAS is lovingly refered to as Anton after the computer in the TV show silicon valley. 

Anton runs ZFS on a Debian system. ZFS is built using a bunch of spinning disks in a JBOD and a bunch of NVMe disks housed in the server. These NVMe disks contain the filesystem metadata as well as a few disks operating as L2 ARC. To share out data, Anton uses SMB with WinBind performing binding to AD. NFS is also used to share out data to various VMs and other Linux systems. For NAS access Anton has a 2 x 25Gb/s NICs which are managed by CTDB. There are also a few containers running on the system.

In the early days on Anton I would just make changes on the fly. Add a script here, tweak a config there. If something upset the system then no major deal becuase users weren't accessing it directly. Well... yeah but surely it would be better to have as close to 100% uptime as possible, right? Surely it would make better sense to handle configs and changes in a much easier way? If for example I was away and someone made a change that damaged the system how can others roll back changes easily? What about new features? Surely it would better if I could test things out. What about in the case of a disater and the whole system needs rebuilding? How can I rebuild quickly. The list can go on.

To solve these problems I started to use SaltStack with git. 

I use three branches in the repo, dev, testing and main. Dev is where all development of salt states and scripts start. When states or scripts have been written I'll merge these changes into the testing branch. Once in testing a GitHub action is triggered to lint any scripts and salt states. Once these tests come back as green I then run a pull command on my salt dev server. From here I then use Jenkins to perform salt calls on a VM thats a mini Anton. With Jenkins I can run a full build of the system or just cherry pick certain parts of Anton to test for example NFS exports or an updated script to handle ZFS snapshots. Once all of the testing is out of the way I'll create a pull request that I can discuss with my collegues. Rather than going in detail over the code as I'm the only salty linuxy jamfy mac guy here, we discuss what the changes will do and how testing went. Once everyone is happy these changes are merged into the main branch. From here I will tag the branch to mark a release. That way we have a nice easy way to roll back changes. 

Next step is to jump onto Anton who runs salt master locally and run a pull on the repo. Since everything has been tested I could just run `salt '*' state.apply test=false` but to be really safe I will always run with `test=true` to fully check whats going to happen. This is also a good check to see if anyone has made changes to the system outside of salt.

And thats it! Adopting a GitOps work flow with salt may seem like a slow processes compared to just getting on with things but it provides a more rounded and stable way of working. Plus I we then have a product that can be rebuilt over and over again to the same state.   
