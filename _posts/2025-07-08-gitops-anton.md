---
title: "Salting GitOps — Managing a NAS with Salt and GitOps"
date: 2025-07-06
categories: [infrastructure, gitops]
tags: [saltstack, gitops, nas, linux]
layout: single
---

In IT, the difference between a “working system” and a resilient system is huge. Early on, I built a backup NAS for our team — affectionately called Anton (named after the computer in Silicon Valley). It started as a side project: ZFS on Debian, JBOD for data with NVMe for metadata and L2 ARC, SMB/NFS shares, CTDB, dual 25 Gb/s networking, and even a few containers.

At first, I made changes directly on the box — a script here, a tweak there. If something broke, it wasn’t the end of the world. But over time, uptime and reliability became just as important as storage capacity. Questions started surfacing:

- How do we roll back changes if something goes wrong?
- How do we test new features without risking production?
- If we ever had to rebuild Anton, how quickly could it be done?

That’s when I moved Anton to a GitOps workflow with SaltStack.

Here’s the flow we use today:

- Branches: Work starts in dev, moves to testing, and finally merges into main.
- Automation: GitHub Actions lint the Salt states and scripts.
- Testing: Jenkins applies changes on a VM that mirrors Anton in a smaller form. We can test everything or just a single piece (e.g. NFS exports).
- Review: Changes are discussed via PRs before merging into main. A tag marks the release for easy rollback.
- Deployment: Anton (running a local Salt master) pulls the latest repo. I always run with test=true first to validate before applying for real.

This approach is slower than logging in and editing configs directly but it’s also safer, repeatable, and rebuildable. If Anton ever went down, we could recreate it in the exact same state from Git.

Moving to GitOps with Salt has been one of the best decisions for reliability and peace of mind. It's also fun :D 
