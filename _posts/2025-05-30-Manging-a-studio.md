---
title: "Managing a Studio with Salt"
date: 2025-05-30
categories: [automation]
tags: [animation, salt]
layout: single
---

1 IT dude vs 60 machines. How on earth do I manage animation software deployments and upgrades to a fleet of render nodes and workstations? How do I keep them all the same? How can I do quickly to reduce down time and keep those renders flowing? SaltStack!

Our studio relies on a wide range of software — from 3ds Max and Blender to VLC and After Effects. Software changes are frequent. Sometimes changes are required between production and post-production. This can be upgrading After Effects or more simply deploying a codec to workstations. Between productions, I often need to update the entire software stack to prepare for the next project. This used to mean a lot of manual installs — time-consuming and prone to mismatch software. But with salt all that has changed! 

When I started at the studio upgrades were handled by a mixture of physically going to the machine, connecting with TeamViewer or using good old RDP. This took a long time. Simple jobs like deploying a new version of a plugin took a few hours but a full upgrade during production downtime could take weeks resulting in artists only being able to render on a few render nodes whilst others were being upgraded.

This wasn't fun! So I set up a tool I had used in a previous job, [SaltStack](https://saltproject.io).

Salt is a configuration and remote management tool that can also use orchestration. Oh, and its open source too! Salt works by having a salt master running on a server and all clients are salt minions. Salt states, YAML config files, are pushed my the master to the salt minions and is then rendered into python code.

Using salt I am able to create state files to handle installs and configurations on my minions at scale. Salt allows me to run my states in parallel across all of my minions at the same time - how cool is that!

I started by creating a Debian VM and setting it up as a Salt master. I then created a GitHub repo to host all of the state files, using a `main` branch for production and a `dev` branch for testing new states. I wrote state files to handle all required installs and configurations. In our studio, we have three core groups of machines: workstations, render nodes, and machines used for a course we run with Leicester College. (These machines also double as render nodes outside teaching hours.) To manage deployments efficiently, I created five 'phase' states — meta-states that include lists of other states to run in order. I then wrote Bash scripts to trigger these phases, with one script for workstations, another for render nodes, and so on.

By running these scripts against minions — individually or as node groups — I can roll out updates at scale or build new machines quickly. Very handy when new render nodes arrive!

Salt has taken several weeks worth of work to upgrade machines and reduced it down to a matter of hours. I simply run my scripts in a `screen` session on my server and walk away. The best bit? I'm using Linux to manage my windows machines :D

