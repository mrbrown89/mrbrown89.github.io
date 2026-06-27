---
title: "saltyBrew"
date: 2026-06-27
categories: [macos, gitops, saltyMacs]
tags: [gitops, saltstack, mdm, launchd, automation]
layout: single
---

Part of my salt deployment includes deploying brew taps, formulae and casks. Salt already has a Homebrew [module](https://saltstack.github.io/docs-saltproject-io/en/latest/ref/modules/all/salt.modules.mac_brew_pkg.html#module-salt.modules.mac_brew_pkg) but there isn't anything for Workbrew.
                                                                                                                  
I wanted a simple way to deploy packages by just updating a pillar file and let salt call Workbrew to install the packages. So I wrote a custom salt [state](https://github.com/mrbrown89/saltyExtensions/blob/main/Workbrew/states/workbrew.py) and [execution](https://github.com/mrbrown89/saltyExtensions/blob/main/Workbrew/modules/workbrew.py) module to handle the deployment of brew packages.

I then created a pillar file:

```
workbrew:
  formulae:
    - nmap
    - mactop
    - awscli
    - ansible
    - docker
    - exiftool
    - iperf3
    - tree
    - go

  casks:
    - ghostty

  tap_packages:
    hashicorp:
      - hashicorp/tap/terraform
      - hashicorp/tap/packer
    cirrus:
      - cirruslabs/cli/tart
    jamf:
      - Jamf-Concepts/tap/jamf-cli
    mrbrown89:
      - mrbrown89/tap/workbrew-cli
      
  trusted_taps:
    - hashicorp/tap
    - cirruslabs/cli
    - mrbrown89/tap
```

From here I create the following salt states:

Formulae:

```
{% set formulae = salt['pillar.get']('workbrew:formulae', []) %}

{% if salt['file.file_exists']('/opt/workbrew/bin/brew') and formulae %}

workbrew_formulae:
  workbrew.installed:
    - pkgs: {{ formulae | yaml }}

{% else %}

workbrew_formulae_skipped:
  test.nop:
    - name: "Workbrew not installed yet or no formulae defined, skipping formulae"

{% endif %}
```

Casks:

```
{% set casks = salt['pillar.get']('workbrew:casks', []) %}

{% if salt['file.file_exists']('/opt/workbrew/bin/brew') and casks %}

workbrew_casks:
  workbrew.installed:
    - pkgs: {{ casks | yaml }}

{% else %}

workbrew_casks_skipped:
  test.nop:
    - name: "Workbrew not installed yet or no casks defined, skipping casks"

{% endif %}
```

Taps:

```
{% set tap_groups = salt['pillar.get']('workbrew:tap_packages', {}) %}
{% set trusted_taps = salt['pillar.get']('workbrew:trusted_taps', []) %}
{% set brew_exists = salt['file.file_exists']('/opt/workbrew/bin/brew') %}

{% if brew_exists and (tap_groups or trusted_taps) %}

# -----------------------------
# Trust taps
# -----------------------------

{% for tap in trusted_taps %}
workbrew_trust_{{ tap | replace('/', '_') }}:
  cmd.run:
    - name: /opt/workbrew/bin/brew trust {{ tap }}
    - unless: /opt/workbrew/bin/brew tap-info --json {{ tap }} 2>/dev/null | jq -e '.[0].trusted == true' >/dev/null
{% endfor %}

# -----------------------------
# Install tapped packages
# -----------------------------

{% for group, pkgs in tap_groups.items() %}
workbrew_{{ group }}_tap_packages:
  workbrew.installed:
    - pkgs: {{ pkgs | yaml }}
    {% if trusted_taps %}
    - require:
{% for tap in trusted_taps %}
      - cmd: workbrew_trust_{{ tap | replace('/', '_') }}
{% endfor %}
    {% endif %}
{% endfor %}

{% else %}

workbrew_taps_skipped:
  test.nop:
    - name: "Workbrew not installed yet or no tap packages/trusted taps defined, skipping tap packages"

{% endif %}
```

There are still a few rough edges, but this already runs in anger across my fleet.
