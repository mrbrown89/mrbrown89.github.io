---
title: "saltyMacs"
date: 2026-05-28
categories: [salt]
tags: [gitops, apple, jamf]
layout: single
---

# saltyMacs

I’ve always been a fan of MDM systems, and I’ve also spent a lot of time using SaltStack to manage both my Linux NAS infrastructure and Windows machines used for animation workloads.

On macOS, my workflow originally relied heavily on MDM policies for things like printer deployment, CIS benchmarking, app installs, and configuration management. While this worked, over time I started feeling that I didn’t have the control I wanted. I saw odd things happen like users loosing printers, policies saying they completed successfully when the hadn’t. 

I wanted something more declarative, version controlled, and easier to reason about.

That led me to build saltyMacs.

saltyMacs is a GitOps-style macOS management system built around:

* Salt running in local mode
* Git as the source of truth
* launchd for scheduling
* a lightweight update and apply loop

Because these Macs are often remote and not permanently connected to a VPN, I decided not to use a traditional Salt master/minion architecture. Instead, every Mac operates independently in masterless mode.

At its core, the workflow is simple:

pull configuration → apply states → log results → repeat

The lifecycle starts with Salt being deployed via MDM alongside a bootstrap script and LaunchDaemon. The Mac then regularly updates its local configuration, applies Salt states, and records the results.

This creates a self managing system where machines continuously enforce their desired state without relying on persistent connectivity to central infrastructure.

Architecture

saltyMacs is built from five main components.

1. Update script

The update script acts as the orchestration layer.

It:

* updates configuration from Git
* prevents duplicate runs
* executes Salt locally
* records execution results
* handles logging and error reporting

2. Git repository

The repository contains:

* Salt states
* pillar data
* app deployment logic
* configuration rules
* naming standards
* custom grains and modules

3. Salt in local mode

Instead of using a central Salt master, saltyMacs uses local execution:

```
salt-call --local state.apply
```

This keeps the system lightweight while still benefiting from Salt’s state engine and declarative configuration model.


4. LaunchDaemon

A LaunchDaemon schedules execution and ensures state enforcement continues automatically after reboots or user logouts.

The Mac effectively becomes self healing over time.

5. Logging and state tracking

Each run records:

* Git activity
* Salt execution output
* state results
* errors and exit codes

This data can then be surfaced back into Jamf using extension attributes and used for workflows, reporting, or troubleshooting. Whilst playing with Fleet I used the reporting feature to use OS Query to pull data from the logs.

Each cycle follows the same sequence:

1. Start update process
2. Check for an existing run
3. Update configuration from Git
4. Apply Salt states
5. Record results
6. Exit cleanly

If another run is already active, the process exits safely instead of overlapping.

saltyMacs helped solve a few issues I kept running into with traditional MDM heavy workflows:

* configuration became fully version controlled
* logic moved out of the MDM layer
* machines became more autonomous
* troubleshooting became easier
* state enforcement became repeatable and predictable

Instead of relying on large numbers of policies and scripts, the Mac continuously converges toward its desired state.

I plan to write a few smaller posts showing how I’ve used saltyMacs to solve specific operational problems.

I’ve also built a smaller companion project called saltyMac, which is designed for quickly spinning up macOS VMs with Tart and applying Salt states for testing and development workflows.
