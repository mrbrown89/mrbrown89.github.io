---
title: "Put the Kettle on. Let's Have a Brew"
date: 2025-06-17
categories: [apple]
tags: [jamf, brew, mac]
layout: single
---

I've been using Homebrew for over a decade on my personal mac. When I started using a mac at work, brew was one of the first things I set up. Being a Jamf user I created a policy that ran a script to install Homebrew to my mac, my mac VMs and test mac. It _kinda_ worked. I had problems with the `$PATH` and I was unable to install brew on my test machines during enrolment with tools like [Jamf Setup Manager](https://GitHub.com/jamf/Setup-Manager) because Homebrew wants to install to a user profile. To install packages I needed I created scripts in Jamf that were pushed out with policies. This just got messy. To solve this problem I started to use Workbrew.

Workbrew is very much like Homebrew but its made for enterprise scale, control and compliance. With Workbrew I can use Jamf to automate installation of Workbrew and deploy packages easily. 

Even better, Workbrew gives me a nice UI where I can:
- See what packages are installed on each device
- Track which versions are running
- Get visibility into known vulnerabilities

Pretty cool, huh! 

To deploy Workbrew I copied the API key install script from Workbrew and created a policy in Jamf to deploy it. I then downloaded the Workbrew package from Workbrew and uploaded it to Jamf. I created a policy to install the package along with the maintenance payload. I set this policy to be trigged with an event. I then edited the API script to run the Jamf policy command with the event flag to trigger the Workbrew package installer. Instead of scoping these policies to a smart group I targeted my work machine only. My work machine lives in a smart group containing all IT computers but we only want my machine to pick up the policies. Workbrew is charged per device so I don't want to end up in a pickle! To better target my machine and any other machine I later add to these policies with package installs I created an extension attribute to find machines with Workbrew installed:

```
#!/bin/bash

if [ -x "/opt/workbrew/bin/brew" ]; then
    echo "<result>Installed</result>"
else
    echo "<result>Not Installed</result>"
fi
```

So to recap, I now have two policies. One to run the API script which needs to run first. The script then triggers another policy to install the Workbrew package. This second policy also runs the maintenance payload to find all devices with Workbrew installed.

Now we need a smart group that I can target to install packages to. Earlier I mentioned my machine is in the IT smart group but I don't want to install Workbrew to every machine in this group unless I later add them. So I created a smart group with the criteria of having the extension attribute of 'Workbrew' as being installed and the device being in the IT smart group. Nice. Now I have a smart group containing devices in the IT smart group that have Workbrew installed. In the future if I need to scale this out to none IT devices i.e. developer macs, I can re-use the extension attribute in the same way against different smart groups. Whoop!

Now I need to get packages installed on my mac. I could manually install packages but I prefer to have thing automated because I'm a nerd and I like automation :D Now, I could create individual scripts to install packages and scope them out to my smart group like I did before but that could get messy when deploying them all in one go plus its a pain to keep creating policies every time I need a new package.

To solve this I created a single script that will curl a script in GitHub and run it. This script will then install any packages I need. My script in Jamf to curl into my GitHub repo is:

```
#!/bin/bash

# GitHub token passed in via Parameter 4 in Jamf
TOKEN="$4"

# URL of your private script
SCRIPT_URL="<put your URL here>"

# Fail if no token
if [ -z "$TOKEN" ]; then
  echo "No GitHub token provided"
  exit 1
fi

# Run the install script using curl + token
echo "Fetching Workbrew install script from GitHub..."
curl -fsSL -H "Authorization: token $TOKEN" "$SCRIPT_URL" | bash
```

I created a policy in Jamf to run the above script on an ongoing basis and scoped it to my IT Workbrew smart group. The script that gets downloaded currently has two functions in it. One to install packages and the other to install casks. The script will sees a list of packages and casks and will first check to see if the package is installed. If not, then Workbrew will install it. 

```
#!/bin/bash

WB="/opt/workbrew/bin/brew"

# Function to install packages only if not already installed
install_pkg() {
    local pkg="$1"
    if ! "$WB" list --formula | grep -q "^$pkg\$"; then
        echo "Installing $pkg..."
        "$WB" install "$pkg"
    else
        echo "$pkg already installed."
    fi
}

install_cask() {
    local app="$1"
    if ! "$WB" list --cask | grep -q "^$app\$"; then
        echo "Installing cask $app..."
        "$WB" install --cask "$app"
    else
        echo "Cask $app already installed."
    fi
}
```

I then put at the bottom of the script `install_pkg <package name>` and `install_cask <cask name>`. 

Now every time the Jamf binary does a policy run anything added to my script on GitHub gets installed! 

What about updates? For that I created another script in Jamf that is set to run on a monthly basis on my IT Workbrew smart group. 

I'm only scratching the surface here so I'm excited to see what else I can do! 


