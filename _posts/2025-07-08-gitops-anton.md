---
title: "Salting GitOps. Managing a NAS with Salt and GitOps"
date: 2025-07-06
categories: [linux]
tags: [gitops, storage, salt]
layout: single
---

We have two NAS systems. One of them is our backup NAS, which I built in-house. This NAS serves several functions. It holds a continually synced copy of all data, performs filesystem snapshots, provides access to data when the main NAS fails, and runs our backup software.

This NAS is affectionately referred to as Anton, named after the computer in the TV show Silicon Valley.

Anton runs ZFS on a Debian system. The ZFS pool is made up of spinning disks in a JBOD, along with several NVMe drives housed within the server. The NVMe disks store filesystem metadata, and some are configured as L2 ARC. For file sharing, Anton uses SMB with WinBind to bind to Active Directory. NFS is also used to share data with various virtual machines and other Linux systems. For network access, Anton is equipped with two 25 Gb/s NICs managed by CTDB. There are also a few containers running on the system.

In the early days of Anton, I would make changes directly on the system. I might add a script here or tweak a configuration there. If something went wrong, it was not a major issue because users were not accessing it directly. However, I began to realise it would be better to aim for as close to 100% uptime as possible - professional pride is at steak here! It also made sense to manage configurations and changes in a more structured way. For example, if I were away and someone made a change that damaged the system, how would others roll back those changes easily? What about testing new features in a safe environment? And in the event of a disaster where the system needed a full rebuild, how could that be done quickly and reliably? These kinds of questions highlighted the need for a better approach.

To address these concerns, I started using SaltStack with Git.

I use three branches in the repository: dev, testing, and main. Development of Salt states and scripts starts in the dev branch. When changes are ready, they are merged into testing. This triggers a GitHub Action that lints the scripts and Salt states. Once these checks pass, I run a pull command on my Salt development server. From there, I use Jenkins to apply Salt states on a virtual machine that replicates Anton in a smaller form. With Jenkins, I can either do a full build of the system or selectively test parts, such as NFS exports or an updated snapshot script.

After successful testing, I create a pull request and discuss it with my colleagues. Once everyone is satisfied, the changes are merged into main. At that point, I tag the branch to mark a release, giving us a clear and easy rollback point.

The next step is to move to Anton, which runs a local Salt master, and pull the updated repository. Since everything has been tested, I could simply run `salt '*' state.apply test=false`, but to be cautious, I always run it with `test=true` first. This double-checks what changes Salt would apply and also helps identify any manual changes made outside of Salt.

That’s the process. Adopting a GitOps workflow with Salt might seem slower than making direct changes, but it creates a much more robust and stable system. It also gives us a repeatable build process that allows Anton to be rebuilt to the exact same state whenever needed.
