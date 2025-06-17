---
title: "Monkeying Around With Munki"
date: 2025-06-08
categories: [apple]
tags: [mac, automation, devops]
layout: single
---

I don't use Munki in my day to day work. Our macs are used onsite and offsite so I use our MDM product, Jamf, to handle software builds. But that doesn't stop me monkeying around (sorry)! 

But for funsies lets think of some interesting scenarios where Munki could work well in the studio I work in. My favourite to think about is a mac based render farm. Due to the nature of our work flow our farm is Windows based with as many cores as we can get and beefy RTX GPUs (very handy when some frames require >90GB of assets!). I manage software deployments on these nodes with SaltStack. I've always dreamed about having some Mac render nodes that can handle some of our work like rendering Adobe AfterEffects jobs. To avoid render issues we need to keep the software versions consistent across workstations and render nodes. Munki would be a great fit to allow me to keep software constancy across a bunch of mac render nodes. Oooh! I could also use Munki to manage software on machines in our galleries (if we had some macs), or I could use Munki to test out software in my VMs before deploying to macs via Jamf, or I could use Munki to... I could go on 😄. 

Lets stop dreaming and start build out a Munki deployment to play with it. For this post I'm using Parallels Pro version 20 on my Apple Silicon Mac. 


## Step 1 - Build Munki Server

For this post I'll be using Ubuntu 24.04.2 LTS.
