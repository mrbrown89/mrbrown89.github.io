---
title: "A Brief Look at Apple’s Gatekeeper"
date: 2025-05-31
categories: [apple, mac, security]
tags: [apple, mac, security]
layout: single
---

Apple has built several layers of security into macOS. The layers are made up of several programs which have evolved over various iterations of macOS. Let’s poke around with one of these programs which is responsible for code signing and download verification - Gatekeeper.

With the release of Mac OS X 10.5 Leopard, Apple introduced File Quarantine. File Quarantine works by adding an extended attribute to files downloaded from the internet. 

When a user attempts to open the file, they are prompted with a warning that the file was downloaded from the internet and if the user is sure they want to continue opening the file. This can be very helpful if a user downloaded what they believe is an image - File Quarantine would display a notification where it’ll say what the user believes is an image is in fact an application. You can view the quarantine attribute by running `xattr` against the file:

```
Matt@Matts-MacBook-Pro:~$ xattr -l /Applications/Thunderbird.app/
com.apple.quarantine: 0183;666f2baa;Safari;F3A898BF-EEDA-4E67-B9A7-FC79FE5B5DDA
```

In the above block we can see some extra info in `0183;666f2baa;Safari;F3A898BF-EEDA-4E67-B9A7-FC79FE5B5DDA`. Let’s break that out. 00083 is the quarantine event and tells the OS not to open it until Gatekeeper checks it. Once the application has been installed you will see this change to `01c3. 666f2baa` is the time stamp the file was downloaded. The time stamp is shown in epoch time (Unix epoch time is the number of seconds since January 1st 1970). `Safari` shows the application that downloaded the file and finally `F3A898BF-EEDA-4E67-B9A7-FC79FE5B5DDA` is the UUID that further identifies the event. All of this information is stored in the quarantine database. 

Let’s dig into this database and query the UUID:

```
Matt@Matts-MacBook-Pro:~$ sqlite3 ~/Library/Preferences/com.apple.LaunchServices.QuarantineEventsV2 
SQLite version 3.43.2 2023-10-10 13:08:14
Enter ".help" for usage hints.
sqlite> select * FROM LSQuarantineEvent WHERE LSQuarantineEventIdentifier="F3A898BF-EEDA-4E67-B9A7-FC79FE5B5DDA";
F3A898BF-EEDA-4E67-B9A7-FC79FE5B5DDA|740254506.21065|com.apple.Safari|Safari||||0|||
```

In the above block, we see the UUID being printed out `F3A898BF-EEDA-4E67-B9A7-FC79FE5B5DDA` which is the LSQuarantineEventIdentifier. We then have `740254506.21065` which is the LSQuarantineTimeStamp - the time stamp. Notice here this is not in the macOS epoch time like in the extended attribute. In the data base, the time stamp is in macOS absolute time. `com.apple.Safari` is the LSQuarantineAgentName - the app name that’s responsible for the quarantine event. `Safari` is the LSQuarantineAgentBundleIdentifier - the bundle name of the application. We then see a bunch of | which are other events in LSQuarantineEvent which are not present in our example. The next bit of information is the 0. This is the LSQuarantineTypeNumber. This will be a number from 0-5 indicating:

0 - Web Download
1 - Other Download
2 - Email Attachment
3 - Message Attachment
4 - Calendar Event Attachment
5 - Other Attachment

The missing events in LSQuarantineEvent in our above example are LSQuarantineSenderName, LSQuarantineSenderAddress (both of which will be populated if the file comes from an email), LSQuarantineOriginTitle, LSQuarantineURLString and finally LSQuarantineOriginAlias.  

Now lets look at the attributes from the same app downloaded with `wget`:

``` 
Matt@Matts-MacBook-Pro:~/Downloads$ xattr -l Thunderbird.app/
Matt@Matts-MacBook-Pro:~/Downloads$
```

`xattr` shows no results. Apple’s security framework that includes the Gatekeeper API appends the attribute to the file. If a program, in this case wget, doesn’t include the API, then the attribute is not added. 

File Quarantine was expanded on in Mac OS X 10.8 Mountain Lion with the introduction of Gatekeeper. Gatekeeper works by checking the digital signature of an application and then performing an action based on the application signature and what the user has set in their security settings for Gatekeeper. Gatekeeper also uses the quarantine attribute assigned by File Quarantine. Users were able to set Gatekeeper to allow applications to install from the Mac App Store, the Mac App Store and trusted developers or from anywhere. In macOS 10.14 Mojave, Apple dropped the anywhere option; however, users are still able to install non-signed apps by either going into settings and clicking on open anyway, by right clicking on the app and clicking on open and then open again in the warning windows, or by disabling Gatekeeper altogether by running sudo `spctl --master-disable`. With the release of macOS 15 Sequoia due to be released in the Autumn, users will no longer be able to use the right-click method. Instead, users will need to go to the privacy and security settings to review the security information before running the application.

Let’s have a closer look at developer signatures:

```
Matt@Matts-MacBook-Pro/Applications: $ spctl -a -t exec -vvv Thunderbird.app/
Thunderbird.app/: accepted
source=Notarized Developer ID
origin=Developer ID Application: Mozilla Corporation (43AQ936H96)
```

Using `spctl` we can query an applications signature. `spctl` shows the `source` as a notarised developer. This means Mozilla has signed Thunderbird with their developer certificate obtained from Apple. Lets have a look at an app downloaded from the Mac App Store:

```
Matt@Matts-MacBook-Pro~/Downloads: $ spctl -a -t exec -vvv /Applications/Safari.app
/Applications/Safari.app: accepted
source=Apple System
origin=Software Signing
```

And now a script that hasn’t been signed with a developers certificate:

```
Matt@Matts-MacBook-Pro/Applications: $ spctl -a -t exec -vvv ~/Downloads/scratch.sh 
/Users/Matt/Downloads/scratch.sh: rejected
source=no usable signature
```

As of macOS 10.15 Catalina, Apple requires all apps to be notarised by Apple even if the app is not distributed by the Mac App Store. Developers are required to upload their application using Apple’s notary service. The service then scans the app for malicious code and if the app passes, a notarisation ticket is issued which we then see as `source=Notarized Developer ID` when looking at an apps signature. The notarisation process is automated. However, if developers want to deploy their apps via the Mac App Store, then their apps must be put through Apple’s App Review, where a human reviews the application to make sure the app is compliant with the App Store guidelines.  

So what does Gatekeeper do with this information? When running `spctl` against Thunderbird, we see the origin show the developer ID as `43AQ936H96`. Gatekeeper will check the gk.db located in `/Library/Apple/System/Library/CoreServices/XProtect.bundle/Contents/Resources/` to see if our developer ID of `43AQ936H96` is in the block list:

```
sqlite> SELECT * FROM blocked_teams WHERE team_id ='43AQ936H96';
sqlite>
```

Cool, we get no response. If we were to get a hit we’d see something like:

```
sqlite> SELECT * FROM blocked_teams WHERE team_id ='ZRT7J747FF';
ZRT7J747FF|0
```

If the developer ID is in the block list, then Gatekeeper and Apple’s XProtect will block the file from running.

Above we have seen how Apple’s Gatekeeper service, in combination with File Quarantine, will append an extended attribute to a downloaded file that has been downloaded by an application that supports Apple’s Gatekeeper API. Gatekeeper will check the file’s signature, which includes the developers ID and Apple notarisation, to make sure it is from a trusted developer. When the executable is run, Gatekeeper will prompt the user, telling them the file has been downloaded from the internet, asking if the user wants to run the executable. If the developer ID is in the block list in the gk.db, then Gatekeeper, along with XProtect, will block the application from running. If, however, a download file has not been notarised by Apple, then the user can override the Gatekeeper by right-clicking on the app and selecting open. 

Like all software, Gatekeeper has not been immune to bugs which has led to threat actors bypassing Gatekeeper controls. Let’s have a look at a few examples. 

Cast your mind back to 2014, LaunchServices didn’t handle file type metadata which allowed a JAR archive to bypass Gatekeeper. This has been given the CVE of CVE-2014-8826.

In 2017, the CVE CVE-2017-2536 was logged, detailing how malicious apps could pretend to be legitimate apps to bypass Gatekeeper

In 2021, analysts saw Shlayer malware exploiting CVE-2021-30657. The attackers bypassed Gatekeeper by exploiting a vulnerability, using a fake notarised app which macOS assumed to be legitimate. 

It is important to acknowledge these issues. While Gatekeeper provides the user with important security features, no-one should underestimate the creativity of those who wish to overcome and circumvent them. 

Nevertheless, one can learn a great deal from how Gatekeeper functions and how malicious actors disrupted its security features. Poking around with command line helps to reveal the process by which apps and downloaded content are made more secure. Even though there has been, and there always will be, security lapses, there is great value in exploring the layers of Apple Security to evaluate the limits of protecting data.
