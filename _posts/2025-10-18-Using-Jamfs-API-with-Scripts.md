---
title: "Using Jamf’s API with Scripts"
date: 2025-10-18
categories: [macos, infrastructure]
tags: [jamf, api, scripting, automation]
layout: single
---

## The Problem

I used to write scripts for Macs on my laptop in either vim or Xcode (yes I know!) and then copy and paste them into the script section of Jamf. If I need to edit the script I used to either copy the script to my laptop, edit and then paste back or simply edit the script directly in Jamf. To add to this I have a Jamf dev and Jamf production instances. I test my scripts in the dev instance and then when happy with them I put the scripts into the production instance.

This works but its a bit of a pain. Having to keep going through menus to paste a script and all that. Surely I could automate this?! I already manage all of my scripts with git and since we use GitHub maybe I could use a GitHub action to automate putting scripts into my Jamf instances.

Enter Jamf’s API!

## Why I Chose the Jamf Pro API

Jamf has two APIs to play with. The classic API and the Jamf Pro API. I chose to use the Jamf Pro API as Jamf seem to encourage the use of this API.

## GitHub Actions Setup and Deploying to Jamf Dev

I tend to have 3 core branches in my repos: dev, testing and main. I start jamming in the dev branch. Just throwing stuff into the editor and start to mould my script to do what I want. Once I have something that I’m happy with and want to test it I merge the scripts into the testing branch.

Once in the testing branch a few jobs are triggered using GitHub actions. In my repo I have the following:

```
├── .github
│   └── workflows
│       ├── deploy-dev-scripts.yml
│       ├── deploy-jamf-scripts.yml
│       └── lint-shell.yml

```
These actions use variables which are stored in GitHub under Settings → Secrets and variables → Actions → Variables.

First a lint job is run which out puts the result to slack:

```
name: Lint Shell Scripts

on:
  push:
    branches: [ testing ]  # Only trigger on dev branch!!!!

jobs:
  lint:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Install ShellCheck
      run: sudo apt-get install -y shellcheck

    - name: Find and Lint Shell Scripts
      run: |
        find . -type f -name "*.sh" | while read -r file; do
          echo "Linting $file"
          shellcheck "$file"
        done

    - name: Send Slack notification
      if: always()
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK }}
        JOB_STATUS: ${{ job.status }}
      run: |
        if [ "$JOB_STATUS" = "success" ]; then
          STATUS="Linting job succeeded"
        else
          STATUS="Linting job failed"
        fi

        curl -X POST -H 'Content-type: application/json' \
          --data "{\"text\":\"${STATUS} in *${GITHUB_REPOSITORY}* on branch *${GITHUB_REF_NAME}*. <${GITHUB_SERVER_URL}/${GITHUB_REPOSITORY}/actions/runs/${GITHUB_RUN_ID}|View run>\"}" \
          "$SLACK_WEBHOOK_URL"

```
The newly created or edited script is then pushed into my Jamf dev instance with the `deploy-dev-scripts` action:

```
name: Deploy Jamf Scripts to dev

on:
  push:
    branches:
      - testing
    paths:
      - 'Scripts/**'
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 2  # Needed to get HEAD~1

      - name: Get Jamf auth token
        id: jamf-auth
        run: |
          set -e
          TOKEN_RESPONSE=$(curl -s -X POST "${{ secrets.JAMF_DEV_URL }}/api/oauth/token" \
            -H "Content-Type: application/x-www-form-urlencoded" \
            --data-urlencode "grant_type=client_credentials" \
            --data-urlencode "client_id=${{ secrets.JAMF_DEV_USERNAME }}" \
            --data-urlencode "client_secret=${{ secrets.JAMF_DEV_PASSWORD }}")

          echo "Token Response: $TOKEN_RESPONSE"

          TOKEN=$(echo "$TOKEN_RESPONSE" | jq -r .access_token)

          if [ -z "$TOKEN" ] || [ "$TOKEN" == "null" ]; then
            echo "Failed to obtain token"
            exit 1
          fi

          echo "token=$TOKEN" >> $GITHUB_OUTPUT

      - name: Deploy or update Jamf scripts
        run: |
          set -x
          TOKEN=${{ steps.jamf-auth.outputs.token }}

          BASE=$(git rev-parse HEAD~1 2>/dev/null || echo "")
          if [ -z "$BASE" ] || ! git rev-parse "$BASE" >/dev/null 2>&1; then
            echo "No valid base ref, using all scripts in Scripts/"
            CHANGED_FILES=$(find Scripts/ -name "*.sh")
          else
            CHANGED_FILES=$(git diff --name-only "$BASE" HEAD -- Scripts/ | grep '\.sh$' || true)
          fi

          echo "Changed files: $CHANGED_FILES"

          if [ -z "$CHANGED_FILES" ]; then
            echo "No .sh script changes to process."
            exit 0
          fi

          for file in $CHANGED_FILES; do
            NAME=$(basename "$file" .sh)
            CONTENT=$(cat "$file")
            META_FILE="devMeta/${NAME}.meta.json"

            if [ -f "$META_FILE" ]; then
              ID=$(jq -r .id "$META_FILE")
              if [[ "$ID" =~ ^[0-9]+$ ]]; then
                echo "Updating existing script ID $ID"
                JSON=$(jq -n \
                  --arg name "$NAME" \
                  --arg content "$CONTENT" \
                  '{name: $name, scriptContents: $content, notes: "Updated via GitHub Actions"}')

                RESPONSE=$(curl -s -X PUT "${{ secrets.JAMF_DEV_URL }}/api/v1/scripts/$ID" \
                  -H "Authorization: Bearer $TOKEN" \
                  -H "Content-Type: application/json" \
                  -d "$JSON")
                echo "$RESPONSE" | jq
              fi
            else
              echo "No meta file — creating new script: $NAME"
              JSON=$(jq -n \
                --arg name "$NAME" \
                --arg content "$CONTENT" \
                '{name: $name, scriptContents: $content, notes: "Created via GitHub Actions"}')

              RESPONSE=$(curl -s -X POST "${{ secrets.JAMF_DEV_URL }}/api/v1/scripts" \
                -H "Authorization: Bearer $TOKEN" \
                -H "Content-Type: application/json" \
                -d "$JSON")

              echo "$RESPONSE" | jq
              NEW_ID=$(echo "$RESPONSE" | jq -r .id)
              if [[ "$NEW_ID" =~ ^[0-9]+$ ]]; then
                echo "Saving new ID $NEW_ID to $META_FILE"
                echo "{ \"id\": $NEW_ID, \"name\": \"$NAME\" }" > "$META_FILE"
              fi
            fi
          done

      - name: Invalidate Jamf token
        run: |
          TOKEN=${{ steps.jamf-auth.outputs.token }}
          curl -s -X POST "${{ secrets.JAMF_DEV_URL }}/api/v1/auth/invalidate-token" \
            -H "Authorization: Bearer $TOKEN"


```
Lets unpack this a little…

1. First we see the trigger:

```
on:
  push:
    branches:
      - testing
    paths:
      - 'Scripts/**'
  workflow_dispatch:
```
Push means the trigger is run automatically when a push is made on the testing branch and when something inside the `Scripts` path has changed. Having `workflow_dispatch` also means we can manually trigger the action in the GitHub UI.

2. Check out repo

```
- uses: actions/checkout@v4
  with:
    fetch-depth: 2
```
This function will check out my repo into the runner. `fetch-depth: 2` ensures I have both `HEAD` and `HEAD~1` so that I can compare file changes.

3. Jamf Token.

Since we are using the Jamf Pro API, we need to grab an API token to use for this runner:

```
      - name: Get Jamf auth token
        id: jamf-auth
        run: |
          set -e
          TOKEN_RESPONSE=$(curl -s -X POST "${{ secrets.JAMF_DEV_URL }}/api/oauth/token" \
            -H "Content-Type: application/x-www-form-urlencoded" \
            --data-urlencode "grant_type=client_credentials" \
            --data-urlencode "client_id=${{ secrets.JAMF_DEV_USERNAME }}" \
            --data-urlencode "client_secret=${{ secrets.JAMF_DEV_PASSWORD }}")

          echo "Token Response: $TOKEN_RESPONSE"

          TOKEN=$(echo "$TOKEN_RESPONSE" | jq -r .access_token)

          if [ -z "$TOKEN" ] || [ "$TOKEN" == "null" ]; then
            echo "Failed to obtain token"
            exit 1
          fi

          echo "token=$TOKEN" >> $GITHUB_OUTPUT
```
Nice! Now we have our token ready for step 4.

4. Finding changes or new files.

```
BASE=$(git rev-parse HEAD~1)
CHANGED_FILES=$(git diff --name-only "$BASE" HEAD -- Scripts/ | grep '\.sh$')
```

The above will see which scripts have been changed in the last commit and if its the first commit then we'll use all script ins `Scripts/`.

5. Now its time to deploy or update scripts in our Jamf dev instance:

```
for file in $CHANGED_FILES; do
  NAME=$(basename "$file" .sh)
  META_FILE="devMeta/${NAME}.meta.json"
```

Wait whats this `META_FILE` bit? When a script is deployed to Jamf an ID is assigned to the script which you can see in the URL when viewing the script in Jamf. If we make a change to the script we need to use the existing script ID to update it. Our action will use a `.meta.json` file to find this information. Once you've deployed the script for the first time, we need to create a meta file. Below is an example:

```
{
  "id": 10,
  "name": "AftermathCollect",
  "filename": "AftermathCollect.sh",
  "categoryId": -1
}
```

The above is in a directory called `devMeta` which you can see in the code block at the start of step 5. This meta file is called `AftermathCollect.meta.json` and you can see the ID of 10 along with the file name `AftermathCollect.sh`. If I update the `AftermathCollect.sh` script, the GitHub action will be able to use the meta file to update the pre existing script in my Jamf dev instance.

6. Clean up:

```
curl -X POST "$JAMF_URL/api/v1/auth/invalidate-token"
```

At the end we run the above to invalidate our API token to be clean and secure. 

## Moving to Production

Pretty cool stuff! Now that I have my scripts in my Jamf dev instance I can deploy them to my test Mac and some test Mac VMs I have. If I need to make a tweak to the script, I jump back into my dev branch, make the change and merge that change into the testing branch which, thanks to the meat file, will update the script in my Jamf dev instance. 

Once I've tested these scripts and I'm happy with them I need to merge my testing branch into the main branch. This will fire off another GitHub action, `deploy-jamf-scripts`, which is the same as my `deploy-dev-scripts` action but instead points to my production Jamf instance. As with the scripts in the testing branch I have meta files that the action will use to update scripts if they already exist. 

Now I can scope these scripts out to devices with confidence... he says... 

To sum up here is a diagram that shows the work flow:

<pre>
dev branch   ───▶   testing branch   ───▶   main branch
    │                     │                     │
    ▼                     ▼                     ▼
local edits   lint and deploy to Jamf Dev   deploy to Jamf Prod
</pre>
