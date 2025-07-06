---
title: "Building a Home macOS GitOps Lab"
date: 2025-07-06
categories: [apple]
tags: [gitops]
layout: single
---

Hook/Intro
- Love macOS so much you want to learn more at home?
- Want to test things out in your own time to explore the how and why?

Why?
- Apple locks things down. So Its fun to tinker
- Hacking is fun! Finding ways to do something that is locked down
- Learn more about macOS admin, configuring macOS and supporting it. 
- Develop your own skill in your own time.
- Transferable skills i.e. to the Linux world.

Build the lab:
- Jenkins container
- Orka Desktop VM
- Parallels VM - snapshots etc
- Cyber Duck
- Git

Testing
- How tests are carried out
- What did we look for?
- What happened?


Problems

- `profiles` tool not working via Jenkins. Example:
```
Started by user matt
[Pipeline] Start of Pipeline
[Pipeline] node
Running on Parallels-macOS in /Users/jenkins/workspace/Disable Airdrop
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Install Dock Profile (.mobileconfig))
[Pipeline] echo
Installing system-level profile...
[Pipeline] sh
+ set -e
+ profile_path=/Users/jenkins/Downloads/disableAirdrop.mobileconfig
+ '[' '!' -f /Users/jenkins/Downloads/disableAirdrop.mobileconfig ']'
+ echo 'Installing with sudo...'
Installing with sudo...
+ sudo /usr/bin/profiles install -type configuration -path /Users/jenkins/Downloads/disableAirdrop.mobileconfig
profiles tool no longer supports installs.  Use System Settings Profiles to add configuration profiles.
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
ERROR: script returned exit code 1
Finished: FAILURE
```

As of BigSur macOS 11 `profiles` has been deprecated. The `man` page for `profiles` states "Starting with macOS 11.0 (profiles tool 8.0 or later), this tool cannot be used to install configuration profiles." This stops us in our tracks. We've hit a wall Apple have put up so what next? If you have access to a sandbox MDM via ork that great but this article is aimed at creating a lab to play in at home so we need to find a way to test our profiles with out an MDM.
         
https://discussions.apple.com/thread/254464992?utm_source=chatgpt.com
https://www.alansiu.net/2021/01/06/semi-automating-profile-installation-in-big-sur/?utm_source=chatgpt.com
Removed the ability for the profiles command to silently install .mobileconfig profiles. … you’ll get an error message: … profiles tool no longer supports installs.”


Whats next
- Further tests
- Simulate custom policy triggers in Jamf
- Build some custom swift code to test things - pushing to devops

