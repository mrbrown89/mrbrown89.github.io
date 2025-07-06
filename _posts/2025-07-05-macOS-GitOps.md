---
title: "macOS GitOps"
date: 2025-07-05
categories: [apple]
tags: [jamf, gitops]
layout: single
---

Over the years at the National Space Centre (NSC) we've done everything manually. We built systems and fixed systems with only the odd thing or two being managed by git. 

This worked to an extent but over time problems developed. 



No quick way to roll back changes
Fixes like scripts made without being saved anywhere which means no easy updates or change has a commit, author, timestamp, and history.
Documentation kept over a sprawl of confluence documents
No automated testing

“Oh, this Jamf script was working last week — what changed?”
With GitOps: Check the Git commit log. Problem found in 30 seconds.

🔐 3. “Manual changes create security risks”

Problem:
An engineer logs into a box and tweaks a setting manually. No record. No approval. Maybe it fixes a bug… or maybe it just created a compliance nightmare.

How GitOps helps:
    •    All changes flow through Git with review and audit.
    •    No “cowboy config changes” — your pipeline becomes the gatekeeper.
    
    
🧪 4. “We can’t test config changes safely”

Problem:
You want to test a policy, script, or package — but the only way to try it is to push it live. Risky.

How GitOps helps:
    •    Encourages dev → test → prod flows, with automation and validation along the way.
    •    You can spin up test VMs, run checks, and only promote when it’s safe.

This is exactly what you’re doing with:

“Push to dev branch → Jenkins tests it in a VM → merge to main when it’s good.”

👥 5. “Too many cooks in the system”

Problem:
Multiple admins are making changes at the same time — and no one knows what the current state should be.

How GitOps helps:
    •    Git becomes the single source of truth.
    •    Branching, pull requests, and reviews reduce conflict and improve collaboration.
    
💥 7. “We can’t recover easily from mistakes”

Problem:
Something breaks. You’re not sure what the last good config was. It takes hours (or days) to rebuild the environment.

How GitOps helps:
    •    Rollbacks are easy — just revert to a previous Git commit.
    •    Rebuilding a machine or config is repeatable from source.


and ultimatley, lack of collaberation.












How did I fix this?
Broke key areas out into separate things and treated them as a product:

- NSCC Software
- Anton
- Jamf
- Switches

Each Product has its own repo in GitHub with branches for dev'ing, testing and production

What tools did we use?
- Jenkins
- Proxmox
- Orka-Desktop
- Containers
- Salt
- Git

What problems did we face and how did we overcome them?

What the future holds?



Sure, writing a quick script or making a small change here and there is a fast way to get out of a mess but do it often enough, and you end up with systems held together by plasters.
Adopting a GitOps workflow results in a carefully implemented IT infrastructure.

Instead of relying on memory or screenshots of web UIs, every script, config, and policy lives in Git. You can review changes, test them in a dev environment, and deploy them knowing you can always roll back.
