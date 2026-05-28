---
title: "Console User Grain"
date: 2026-05-28
categories: [macos, gitops, infrastructure]
tags: [gitops, saltstack, mdm, launchd, automation, saltyMacs]
layout: single
---


# saltyMacs — Console User Grain

During my Jamf 400 course one of the tasks we had to do was to find the console user with some code:

```
scutil <<< "show State:/Users/ConsoleUser" | awk '/Name :/ && ! /loginwindow/ { print $3 }'
```

Salt doesn’t have a built in grain for the active macOS console user, so I wrote a small Python grain to retrieve it:

```
import subprocess


def console_user():
    """
    Return the currently logged in console user on macOS.
    """

    try:
        user = subprocess.check_output(
            ["/usr/bin/stat", "-f%Su", "/dev/console"],
            text=True
        ).strip()

        return {
            "console_user": user
        }

    except Exception:
        return {
            "console_user": None
        }
```

Ah! But that is different to the bash code example I gave earlier. I used `stat` as it’s not dependent on parsing text output like the bash example. It's also one system call keeping the grain small and simple.

The grain uses `/dev/console`, which represents the active macOS console session. By checking the ownership of that device we can determine the active GUI console user.

You can find the grain and a little README in my [saltyExtensions](https://github.com/mrbrown89/saltyExtensions/tree/main/consoleUser/grains) repo.

I use this grain on my personal mac for installing packages via brew for example here is part of my brew state file that handles tap installations:

```
{% set brew = pillar.get('brew', {}) %}
{% set brew_bin = '/opt/homebrew/bin/brew' %}
{% set console_user = grains['console_user'] %}

# -------------------------------------------------
# Homebrew taps
# -------------------------------------------------

{% for tap in brew.get('taps', []) %}

brew_tap_{{ tap | replace('/', '_') }}:
  cmd.run:
    - name: {{ brew_bin }} tap {{ tap }}
    - unless: {{ brew_bin }} tap | grep -q "^{{ tap }}$"
    - runas: {{ console_user }}

{% endfor %}
```

You can see at the start of the state I declare `console_user` using the grain:

```
{% set console_user = grains['console_user'] %}
```

