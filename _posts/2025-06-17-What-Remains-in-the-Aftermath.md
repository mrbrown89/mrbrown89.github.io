---
title: "What Remains in the Aftermath"
date: 2025-06-17
categories: [apple]
tags: [jamf, jamfprotect, security, mac]
layout: single
---

At Jamf Nation Live in London 2024 I attended a talk that mentioned [Jamf Aftermath](https://github.com/jamf/aftermath). I have a habit of getting over excited about these sorts of things so when I got back to work I tried it out.

What is aftermath? Aftermath is a Jamf tool that automates the collection of forensic data from a Mac after a suspected compromise—perfect for IR teams and IT admins. 

You can deploy Aftermath via MDM or install it manually. You can then trigger Aftermath manually or using a script in your MDM. I came across the [SOAR playbook](https://github.com/jamf/jamfprotect/tree/main/soar_playbooks) on the Jamf Protect GitHub [repo](https://github.com/jamf/jamfprotect) and thought I'd give it a go! Here's our work flow:

1. Analytic Triggeredd
 - The `high` file marker will be added to `/Library/Application Support/JamfProtect/groups/`
 - At a Jamf recon scan, the marker will be picked up with our extension attribute script that we deployed with our `Jamf Protect - Smart Group` extended attribute template.
 - Devices that have the extended attribute will be added to our `Jamf Protect: Aftermath` smart group.
 
 2. Trigger Aftermath Scan
  - A policy containing a script scoped to the `Jamf Protect: Aftermath` smart group will be run.
  - The script will run Aftermath and dump the data to `/tmp`
  - Script will trigger another policy with a custom event of `am_collect`.
  
  3. Aftermath Collect
  - The custom policy event of `am_collect` at the end of step 2 triggers a policy to grab aftermath data and upload it to AWS. The script will download and install the AWS binary if needed.
  - To upload to AWS the above script triggers another policy with the custom event `aws_creds` to download our AWS credentials from Jamf.
  - Our collect script will finish off by uninstalling the AWS binary and deleting our file marker in `/Library/Application Support/JamfProtect/groups/`.
  
Lets break this down...

## 1. Analytic Triggered

Following the [Aftermath Collection](https://github.com/jamf/jamfprotect/tree/main/soar_playbooks/aftermath_collection) playbook I first deployed Aftermath to all my machines with a policy. I then went through all of the analytics in Jamf Protect that I want to use Aftermath with i.e. if the analytic is triggered then the Aftermath collection is run. To do this I selected the tick box `Add to Jamf Pro Smart Group` under each analytic I wanted and entered a name for attribute. As per the SOAR documentation I used `high`. 

Once I edited all of the analytics (it took a while!) I popped back into Jamf Pro and created a smart group called `Jamf Protect: Aftermath` with the criteria of `Jamf Protect - Smart Group`, operator of `like` and the value of `high`. Nice! So now when an analytic is triggered, Jamf Protect will create a marker file called `high` in `/Library/Application Support/JamfProtect/groups/`. So what's the extension attribute script? This is in the templates section so you simply need to pick the `Jamf Protect - Smart Groups`.

Cool, so we now have Aftermath deployed to our macs, our analytics marked with the value of `high` and a smart group called `Jamf Protect: Aftermath` which has the criteria of `Jamf Protect - Smart Group`. If any machine triggers one of these analytics then the device will become a member of the smart group.

## 2. Trigger Aftermath Scan

First we need to create our AWS bucket to send data to. Now we need a script to run Aftermath and then trigger the next policy in our workflow. 

```
#!/bin/bash

/usr/local/bin/aftermath --pretty; /usr/local/bin/jamf policy -event am_collect
```
Notice here that after running Aftermath we run `jamf policy -event am_collect` which will trigger our next policy.

## 3. Aftermath Collect

We need to copy the `aws_s3` [script](https://github.com/jamf/jamfprotect/blob/main/soar_playbooks/aftermath_collection/aws_s3/aftermath_collection.sh) from the SOAR playbook and paste it into scripts in Jamf Pro. Make sure to edit the script variables for your S3 bucket and the Jamf Protect Analytic smartgroup identifier to reflect the one you set (in this case `high`). We also need to create a package containing our AWS credentials. We need to create a policy to run this script with the custom trigger of `am_collect`. We also need to create a policy to deploy our AWS credentials with the custom trigger of `aws_creds` which is triggered in our `aws_s3` script. 

So in this step, our `aws_s3` script is triggered by the custom trigger `am_collect` which is triggered at the end of our script in step 2. As part of its run the `aws_s3` will download the AWS binary, trigger another Jamf policy with the trigger of `aws_creds` to download our AWS credentials, upload the Aftermath data to AWS, uninstall AWS and finally delete the file marker in `/Library/Application Support/JamfProtect/groups/` followed by a Jamf recon which will remove our device from the `Jamf Protect: Aftermath` smart group.

Nice!

It wasn't plain sailing during my deployment, however. I was having a right nightmare trying to upload my data to AWS due to environment and permissions. I had to make a few edits to the script.

I explicitly set the path to the AWS CLI binary `/usr/local/bin/aws` instead of relying on `$PATH`, which behaves inconsistently when scripts run as root via Jamf. I also fixed credential handling — the original script assumed credentials were in the user’s `~/.aws`, which fails under root. I changed it to pull credentials from `/opt/.aws` and copy them into `/var/root/.aws`, where the AWS CLI expects them in root context.



## Bonus Points

If you use Jamf Security Cloud, why not utilise that to lock the internet down on the infected device? You can scope our smart group to a restricted group in Security Cloud. Keep in mind though that this smart group is created using an extension attribute which is deleted when the Aftermath collection script is run so make sure to remove or comment out:

```
    # Remove Jamf Protect extension attribute if needed
    if [[ -n "$analyticEA" ]]; then
        echo "Cleaning up extension attribute: $analyticEA"
        rm "/Library/Application Support/JamfProtect/groups/$analyticEA"
    fi
```

Or just not fill in the `analyticEA` variable. Either way 😁


So how do we test all of this? Simple! Use [Jamf's Check](https://marketplace.jamf.com/details/jamfcheck) application. From here you can trigger analytics and watch the workflow. It's awesome!
