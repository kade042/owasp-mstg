## Basic Security Testing on iOS

### Foreword on Swift and Objective-C

Vast majority of this tutorial is relevant to applications written mainly in Objective-C or having bridged Swift types. Please note that these languages are fundamentally different. Features like method swizzling, which is heavily used by Cycript will not work with Swift methods. At the time of writing of this testing guide, Frida does not support instrumentation of Swift methods.

### Setting Up Your Testing Environment

In contrast to the Android emulator, which fully emulates the processor and hardware of an actual Android device, the simulator in the iOS SDK offers a higher-level *simulation* of an iOS device.  Most importantly, emulator binaries are compiled to x86 code instead of ARM code. Apps compiled for an actual device don't run, making the simulator completely useless for black-box-analysis and reverse engineering.

Ideally you want to have a jailbroken iPhone or iPad available for running tests. That way, you get root access to the device and can install a variety of useful tools, making the security testing process easy. If you don't have access to a jailbroken device, you can apply the workarounds described later in this chapter, but be prepared for a less smooth experience.

For your mobile app testing setup you should have at least the following:

- Laptop with admin rights, VirtualBox with Kali Linux
- WiFi network with client to client traffic permitted (multiplexing through USB is also possible)
- At least one jailbroken iOS device (with desired iOS version)
- Burp Suite or other web proxy

If you want to get serious with iOS security testing, you need a Mac, for the simple reason that XCode and the iOS SDK are only available for Mac OS. Many tasks that you can do effortlessly on Mac are a chore, or even impossible, on Windows and Linux. The following setup is recommended:

- Macbook with XCode and Developer's Profile
- WiFi network as previously
- At least two iOS devices, one jailbroken, second non-jailbroken
- Burp Suite or other web proxy
- Hopper or IDA Pro

#### Jailbreaking the iOS Device

iOS jailbreaking is often compared to Android rooting. Actually, we have three different things here and it is important to clearly distinguish them.

On the Android side we have:

- Rooting: this typically consists of installing the `su` binary within the existing system or replacing the whole system with an already rooted custom ROM. Normally, exploits are not required in order to obtain root access.
- Flashing custom ROMs (that might be already rooted): allows to completely replace the OS running on the device after unlocking the bootloader (which might require an exploit). There is no such thing on iOS as it is closed-source and _thanks_ to the bootloader that only allows Apple-signed images to be booted and flashed (which is also the reason why downgrades/upgrades to not-signed-by-Apple iOS images are not possible).

On iOS side we have:

- Jailbreaking: colloquially, the word "jailbreak" if often used to refer to all-in-one tools that automate the complete jailbreaking progress, from executing the exploit(s) to disable system protections (such as Apple's code signing mechanisms) and install the Cydia app store. If you're planning to do any form of dynamic security testing on an iOS device, you'll have a much easier time on a jailbroken device, as most useful testing tools are only available outside the App Store.

Developing a jailbreak for any given version of iOS is not an easy endeavor. As a security tester, you'll most likely want to use publicly available jailbreak tools (don't worry, we're all script kiddies in some areas). Even so, we recommend studying the techniques used to jailbreak various versions of iOS in the past - you'll encounter many highly interesting exploits and learn a lot about the internals of the OS. For example, Pangu9 for iOS 9.x [exploited at least five vulnerabilities](https://www.theiphonewiki.com/wiki/Jailbreak_Exploits), including a use-after-free bug in the kernel (CVE-2015-6794) and an arbitrary file system access vulnerability in the Photos app (CVE-2015-7037).

#### Types of Jailbreaking Methods

In jailbreak lingo, we talk about tethered and untethered jailbreaking methods. In the "tethered" scenario, the jailbreak doesn't persist throughout reboots, so the device must be connected (tethered) to a computer during every reboot to re-apply it. "Untethered" jailbreaks need only be applied once, making them the most popular choice for end users.

#### Benefits of Jailbreaking

A standard user will want to jailbreak in order to tweak the iOS system appearance, add new features or install _free_ third party apps. However, for a security tester the benefits of jailbreaking an iOS Device go far beyond simply tweaking the system. They include but are not limited to the following:

- Removing the part of the security (and other) limitations on the OS imposed by Apple
- Providing root access to the operating system
- Allowing applications and tools not signed by Apple to be installed and run without any restrictions
- Debugging and performing dynamic analysis
- Providing access to the Objective-C Runtime

#### Caveats and Considerations about Jailbreaking

Jailbreaking iOS devices is becoming more and more complicated as Apple keeps hardening the system and patching the corresponding vulnerabilities that jailbreaks are based on. Additionally, it has become a very time sensitive procedure as they stop signing these vulnerable versions within relative short time intervals (unless they are hardware-based vulnerabilities). This means that, contrary to Android, you can't downgrade iOS version with one exception explained below, which naturally creates a problem, when there is a major bump in iOS version (e.g. from 9 to 10) and there is no public jailbreak for the new OS.

A recommendation here is: if you have a jailbroken device that you use for security testing, keep it as is, unless you are 100% sure that you can perform a newer jailbreak to it. Additionally you can think of having a second one, which will be updated with every major iOS release and wait for public jailbreak to be released. Once a public jailbreak is released, Apple is quite fast in releasing a patch, hence you have only a couple of days to upgrade to the newest iOS version and jailbreak it (if upgrade is necessary).

The iOS upgrade process is performed online and is based on a challenge-response process. The device will perform the OS installation only if the response to the challenge is signed by Apple. This is what researchers call 'signing window' and explains the fact that you can't simply store the OTA firmware package downloaded via iTunes and load it to the device at any time. During minor iOS upgrades, it is possible that two versions are signed at the same time by Apple. This is the only case when you can possibly downgrade iOS version. You can check current signing window and download OTA Firmwares from the [IPSW Downloads website](https://ipsw.me).


#### How to Jailbreak iOS?

Jailbreaking methods vary across iOS versions. Best choice is to [check if a public jailbreak is available for your iOS version](https://canijailbreak.com/). Beware of fake tools and spyware that is often distributed around the Internet, often hiding behind domain names similar to the jailbreaking group/author.

Let's say you have a device running iOS 9.0, for this version you'll find a jailbreak (Pangu 1.3.0), at least for 64 bit devices. In the case that you have another version for which there's not a jailbreak available, you could still jailbreak it if you downgrade/upgrade to the target _jailbreakable_ iOS version (via IPSW download and iTunes). However, this might not be possible if the required iOS version is not signed anymore by Apple.

The iOS jailbreak scene evolves so rapidly that it is difficult to provide up-to-date instructions. However, we can point you to some, at the time of this writing, reliable sources:

- [The iPhone Wiki](https://www.theiphonewiki.com/)
- [Redmond Pie](http://www.redmondpie.com/)
- [Reddit Jailbreak](https://www.reddit.com/r/jailbreak/)

Note that obviously OWASP and the MSTG will not be responsible if you end up bricking your iOS device!

#### Dealing with Jailbreak Detection

Some apps attempt to detect whether the iOS device they're installed on is jailbroken. The reason for this jailbreaking deactivates some of iOS' default security mechanisms, leading to a less trustable environment.

The core dilemma with this approach is that, by definition, jailbreaking causes the app's environment to be unreliable: The APIs used to test whether a device is jailbroken can be manipulated, and with code signing disabled, the jailbreak detection code can easily be patched out. It is therefore not a very effective way of impeding reverse engineers. Nevertheless, jailbreak detection can be useful in the context of a larger software protection scheme. We'll revisit this topic in the next chapter.

### Preparing the Test Environment

![Cydia Store](Images/Chapters/0x06b/cydia.png "Cydia Store")

Once you have your iOS device jailbroken and Cydia is installed (as per screenshot), proceed as following:

1. From Cydia install aptitude and openssh
2. SSH to your iDevice
  * Two users are `root` and `mobile`
  * Default password is `alpine`
3. Add the following repository to Cydia: `https://build.frida.re`
4. Install Frida from Cydia
5. Install following packages with aptitude

```
inetutils
syslogd
less
com.autopear.installipa
class-dump
com.ericasadun.utilities
odcctools
cycript
sqlite3
adv-cmds
bigbosshackertools
```

Your workstation should have SSH client, Hopper Disassembler, Burp and Frida installed. You can install Frida with pip, for instance:

```
$ sudo pip install frida
```

#### SSH Connection via USB

As per the normal behavior, iTunes communicates with the iPhone via the <code>usbmux</code>, which is a system for multiplexing several "connections" over one USB pipe. This system provides a TCP-like system where multiple processes on the host machine open up connections to specific, numbered ports on the mobile device.

[usbmuxd](https://github.com/libimobiledevice/usbmuxd) is a socket daemon that watches for iPhone connections via USB. You can use it to map listening localhost sockets from the mobile device to TCP ports on your host machine. This conveniently allows you to SSH into your device independent of network settings. When it detects an iPhone running in normal mode, it will connect to it and then start relaying requests that it receives via /var/run/usbmuxd.

On MacOS:

```
$ brew install libimobiledevice
$ iproxy 2222 22
$ ssh -p 2222 root@localhost
iPhone:~ root#
```

Python client:

```bash
$ ./tcprelay.py -t 22:2222
$ ssh -p 2222 root@localhost
iPhone:~ root#
```

### Typical iOS Application Test Workflow

Typical workflow for iOS Application test is following:

1. Obtain IPA file
2. Bypass jailbreak detection (if present)
3. Bypass certificate pinning (if present)
4. Inspect HTTP(S) traffic - usual web app test
5. Abuse application logic by runtime manipulation
6. Check for local data storage (caches, binary cookies, plists, databases)
7. Check for client-specific bugs, e.g. SQLi, XSS
8. Other checks like: logging to ASL with NSLog, application compile options, application screenshots, no app backgrounding

### Static Analysis

<!-- #### With Source Code -->

<!-- TODO [Add content on security Static Analysis of an iOS app with source code] -->

#### Without Source Code

##### Folder Structure

System applications can be found in "/Applications". For user-installed apps, you can use [IPA Installer Console](http://cydia.saurik.com/package/com.autopear.installipa) to navigate to the appropriate folders:

```
iOS8-jailbreak:~ root# installipa -l
me.scan.qrcodereader
iOS8-jailbreak:~ root# installipa -i me.scan.qrcodereader
Bundle: /private/var/mobile/Containers/Bundle/Application/09D08A0A-0BC5-423C-8CC3-FF9499E0B19C
Application: /private/var/mobile/Containers/Bundle/Application/09D08A0A-0BC5-423C-8CC3-FF9499E0B19C/QR Reader.app
Data: /private/var/mobile/Containers/Data/Application/297EEF1B-9CC5-463C-97F7-FB062C864E56
```

As you can see, there are three main directories: <codde>Bundle</code>, <code>Application</code> and <code>Data</code>. The <code>Application</code> directory is simply a subdirectory of Bundle.
The static installer files are located in Application, whereas all user data resides in the Data directory.

The random string in the URI is application's GUID, which will be different from installation to installation.

### Dynamic Analysis

#### Monitoring Console Logs

Many apps log informative (and potentially sensitive) messages to the console log. Besides that, the log also contains crash reports and potentially other useful information. You can collect console logs through the XCode "Devices" window as follows:

1. Launch Xcode
2. Connect your device to your host computer
3. Choose Devices from the Window menu
4. Click on your connected iOS device in the left section of the Devices window
5. Reproduce the problem
6. Click the triangle in a box toggle located in the lower-left corner of the right section of the Devices
window to expose the console log contents

To save the console output to a text file, click the circle with a downward-pointing arrow at the bottom right.

![Console logs](Images/Chapters/0x06b/device_console.jpg "Monitoring console logs through XCode")

#### Setting up a Web Proxy using BurpSuite

Burp Suite is an integrated platform for performing security testing of mobile and web applications. Its various tools work seamlessly together to support the entire testing process, from initial mapping and analysis of an application’s attack surface, to finding and exploiting security vulnerabilities. It is a toolkit where Burp proxy operates as a web proxy server, and sits as a man-in-the-middle between the browser and web server(s). It allows the interception, inspection and modification of the raw HTTP traffic passing in both directions.

Setting up Burp to proxy your traffic through is pretty straightforward. It is assumed that you have both: iDevice and workstation connected to the same WiFi network where client to client traffic is permitted. If client-to-client traffic is not permitted, it is possible to use usbmuxd in order to connect to Burp through USB.

Portswigger also provides a good [tutorial on setting up an iOS Device to work with Burp](https://support.portswigger.net/customer/portal/articles/1841108-configuring-an-ios-device-to-work-with-burp "Configuring an iOS Device to Work With Burp").

###### Configure the Burp Proxy Listener
- In Burp, go to the “Proxy” tab and then the “Options” tab.
- In the “Proxy Listeners" section, click the “Add” button.
- In the "Binding" tab, in the “Bind to port:” box, enter a port number that is not currently in use, e.g. “8082”.
- Then select the “All interfaces” option, and click "OK".
- The Proxy listener should now be configured and running.

###### Configure Your Device to Use the Proxy
- In your iOS device, go to the “Settings” menu.
- Tap the “Wi-Fi” option from the "Settings" menu.
- Tap the “i” (information) option next to the name of your network.
- Under the "HTTP PROXY" title, tap the “Manual” tab.
- In the "Server" field, enter the IP address of the computer that is running Burp.
- In the “Port” field, enter the port number configured in the “Proxy Listeners” section earlier, in this example “8082”.

###### Test the Burp configuration for HTTP Requests
- In Burp, go to the "Proxy Intercept" tab, and ensure that intercept is “on”.
- Open the browser on your iOS device and go to an HTTP web page. 
- The request should be intercepted in Burp.

###### Installing Burp's CA Certificate in an iOS Device:
- With Burp running, visit http://burp in your browser and click the “CA Certificate” link to download and install your Burp CA certificate.

###### Test the Burp configuration for HTTPS Requests
- In Burp, go to the "Proxy Intercept" tab, and ensure that intercept is “on”.
- Open the browser on your iOS device and go to an HTTPs web page. 
- The request should be intercepted in Burp.

#### Setting up a Web Proxy using OWASP ZAP

-- TODO

#### Bypassing Certificate Pinning

When you try to intercept the mobile app and server communication you might fail due to certificate pinning. Certificate Pinning is a practice used to tighten security of TLS connection. When an application is connecting to the server using TLS, it checks if the server's certificate is signed with trusted CA's private key. The verification is based on checking the signature with public key that is within device's key store. This in turn contains public keys of all trusted root CAs.

Certificate pinning means that our application will have server's certificate or hash of the certificate hardcoded into the source code.
This protects against two main attack scenarios:

- Compromised CA issuing certificate for our domain to a third-party
- Phishing attacks that would add a third-party root CA to device's trust store

The simplest method is to use `SSL Kill Switch` (can be installed via Cydia store), which will hook on all high-level API calls and bypass certificate pinning. There are some cases, though, where certificate pinning is more tricky to bypass. Things to look for when you try to bypass certificate pinning are:

- following API calls: `NSURLSession`, `CFStream`, `AFNetworking`
- during static analysis, try to look for methods/strings containing words like 'pinning', 'X509', 'Certificate', etc.
- sometimes, more low-level verification can be done using e.g. openssl. There are tutorials [20] on how to bypass this.
- some dual-stack applications written using Apache Cordova or Adobe Phonegap heavily use callbacks. You can look for the callback function called upon success and call it manually with Cycript
- sometimes the certificate resides as a file within application bundle. It might be sufficient to replace it with Burp's certificate, but beware of certificate's SHA sum that might be hardcoded in the binary. In that case you must replace it too!

Certificate pinning is a good security practice and should be used for all applications handling sensitive information. [EFF's Observatory](https://www.eff.org/pl/observatory) provides list of root and intermediate CAs that are by default trusted on major operating systems. Please also refer to a [map of the 650-odd organizations that function as Certificate Authorities trusted (directly or indirectly) by Mozilla or Microsoft](https://www.eff.org/files/colour_map_of_CAs.pdf "Map of the 650-odd organizations that function as Certificate Authorities trusted (directly or indirectly) by Mozilla or Microsoft"). Use certificate pinning if you don't trust at least one of these CAs.

If you want to get more details on white-box testing and usual code patters, refer to iOS Application Security by David Thiel. It contains description and code snippets of most-common techniques used to perform certificate pinning.

To get more information on testing transport security, please refer to section "Testing Network Communication".

#### Dynamic Analysis On Jailbroken Devices

Life is easy with a jailbroken device: Not only do you gain easy access to the app's sandbox, you can also use more powerful dynamic analysis techniques due to the lack of code singing. On iOS, most dynamic analysis tools are built on top of Cydia Substrate, a framework for developing runtime patches that we will cover in more detail in the "Tampering and Reverse Engineering" chapter. For basic API monitoring purposes however, you can get away without knowing Substrate in detail - you can simply use existing tools built for this purpose.

##### Copying App Data Files

Files belonging to an app are stored app's data directory. To identify the correct path, ssh into the device and retrieve the package information using IPA Installer Console:

```bash
iPhone:~ root# ipainstaller -l
sg.vp.UnCrackable-2
sg.vp.UnCrackable1

iPhone:~ root# ipainstaller -i sg.vp.UnCrackable1
Identifier: sg.vp.UnCrackable1
Version: 1
Short Version: 1.0
Name: UnCrackable1
Display Name: UnCrackable Level 1
Bundle: /private/var/mobile/Containers/Bundle/Application/A8BD91A9-3C81-4674-A790-AF8CDCA8A2F1
Application: /private/var/mobile/Containers/Bundle/Application/A8BD91A9-3C81-4674-A790-AF8CDCA8A2F1/UnCrackable Level 1.app
Data: /private/var/mobile/Containers/Data/Application/A8AE15EE-DC8B-4F1C-91A5-1FED35258D87
```

You can now simply archive the data directory and pull it from the device using scp.

```bash
iPhone:~ root# tar czvf /tmp/data.tgz /private/var/mobile/Containers/Data/Application/A8AE15EE-DC8B-4F1C-91A5-1FED35258D87
iPhone:~ root# exit
$ scp -P 2222 root@localhost:/tmp/data.tgz .
```

##### Dumping KeyChain Data

[Keychain-Dumper](https://github.com/ptoomey3/Keychain-Dumper/) lets you dump the contents of the KeyChain on a jailbroken device. The easiest way of running the tool is to download the binary from its GitHub repo:

``` bash
$ git clone https://github.com/ptoomey3/Keychain-Dumper
$ scp -P 2222 Keychain-Dumper/keychain_dumper root@localhost:/tmp/
$ ssh -p 2222 root@localhost
iPhone:~ root# chmod +x /tmp/keychain_dumper
iPhone:~ root# /tmp/keychain_dumper

(...)

Generic Password
----------------
Service: myApp
Account: key3
Entitlement Group: RUD9L355Y.sg.vantagepoint.example
Label: (null)
Generic Field: (null)
Keychain Data: SmJSWxEs

Generic Password
----------------
Service: myApp
Account: key7
Entitlement Group: RUD9L355Y.sg.vantagepoint.example
Label: (null)
Generic Field: (null)
Keychain Data: WOg1DfuH
```

Note however that this binary is signed with a self-signed certificate with a "wildcard" entitlement, granting access to *all* items in the Keychain - if you are paranoid, or have highly sensitive private data on your test device, you might want to build the tool from source and manually sign the appropriate entitlements into your build - instructions for doing this are available in the GitHub repository.

##### Security Profiling with Introspy

Introspy is an open-source security profiler for iOS released by iSecPartners. Built on top of Substrate, it can be used to log security-sensitive API calls on a jailbroken device. The recorded API calls are sent to the console and written to a database file, which can then be converted into an HTML report using Introspy-Analyzer. 

After successful installation of IntroSpy, respring the iOS device and follow the below steps: 

- Go to Settings and you will see a section for Introspy.
- There should be two new menus in the device Settings. 
	- Introspy - Apps: The Apps menu allows you to select which applications will be profiled 
	- Introspy - Settings: The Settings menu defines which API groups are being hooked
- Now, kill and restart the app you want to monitor 
- Go to General > Settings > Introspy - Apps > Select the target app

Once the target app has been selected, make sure it is running. If it is running, quit it and restart the app again. Make sure that your device is connected to your computer as we want to see the device logs that the Introspy analyzer will be logging. Also, open Xcode on your machine, go to Window -> Devices. Choose your device from the menu on the left and select Console. You will now be able to see the device logs while browsing the application. Please note that analyzer will work in the background to collect the information.

###### Generating HTML Reports with Introspy 

- The tracer will store data about API calls made by applications in a database stored on the device (actually one in each folder). This database can be fed to a Python script called Introspy-Analyzer in order to generate HTML reports that make it a lot easier to review the data collected by the tracer. The script will also analyze and flag dangerous API calls in order to facilitate the process of identifying vulnerabilities within iOS applications. 

- Apart from displaying the runtime  information about the app in the Console, Introspy also saves it in a sqlite database file on your device. Introspy consists of 2 modules, the Tracer and the Analyzer.

- We can use the Tracer to perform runtime analysis of the application. The tracer can then store the results in a sqlite file which can be later  used by the analyzer for analysis, or it can also just log all the data to the device console. The Analyzer can also generate a well detailed HTML report from the database file.

To generate HTML report, follow the below steps: 

1. Install Introspy analyzer on computer:
	1. As a package ```pip install git+https://github.com/iSECPartners/Introspy-Analyzer.git```
	2. Or locally, ```git clone https://github.com/iSECPartners/Introspy-Analyzer.git```
2. Databases can be fetched directly by Introspy-Analyzer over SSH by running following command: 

	`python -m introspy -p ios -o output_dir -f <iOSDEVICE_IP>`
3. Introspy will ask you to select a database file. These database files are created for each application that we had selected from the Settings. Select the database for the target app 
4. The database file will be saved in the present directory as well as a folder with the name Target-Report will be created
5. Navigate to output folder and open report.html. Introspy displays the complete information in a much more presentable format. We can see the list of traced calls along with the arguments that were passed. 

- References 
	1. [Introspy](https://github.com/iSECPartners/Introspy-iOS)
	2. [Introspy Analyzer](https://github.com/iSECPartners/Introspy-Analyzer)

#### Dynamic Analysis on Non-Jailbroken Devices

If you don't have access to a jailbroken device, you can patch and repackage the target app to load a dynamic library at startup. This way, you can instrument the app and can do pretty much everything you need for a dynamical analysis (of course, you can't break out of the sandbox that way, but you usually don't need to). This technique however works only on if the app binary isn't FairPlay-encrypted (i.e. obtained from the app store).

Thanks to Apple's confusing provisioning and code signing system, re-signing an app is more challenging than one would expect. iOS will refuse to run an app unless you get the provisioning profile and code signature header absolutely right. This requires you to learn about a whole lot of concepts - different types of certificates, BundleIDs, application IDs, team identifiers, and how they are tied together using Apple's build tools. Suffice it to say, getting the OS to run a particular binary that hasn't been built using the default way (XCode) can be an daunting process.

The toolset we're going to use consists of optool, Apple's build tools and some shell commands. Our method is inspired by the resign script from [Vincent Tan's Swizzler project](https://github.com/vtky/Swizzler2/). An alternative way of repackaging using different tools was [described by NCC group](https://www.nccgroup.trust/au/about-us/newsroom-and-events/blogs/2016/october/ios-instrumentation-without-jailbreak/ "NCC blog - iOS instrumentation without jailbreak") .

To reproduce the steps listed below, download [UnCrackable iOS App Level 1](https://github.com/OWASP/owasp-mstg/tree/master/OMTG-Files/02_Crackmes/02_iOS/UnCrackable_Level1) from the OWASP Mobile Testing Guide repo. Our goal is to make the UnCrackable app load FridaGadget.dylib during startup so we can instrument it using Frida.

##### Getting a Developer Provisioning Profile and Certificate

The *provisioning profile* is a plist file signed by Apple that whitelists your code signing certificate on one or multiple devices. In other words, this is Apple explicitly allowing your app to run in certain contexts, such as debugging on selected devices (development profile). The provisioning profile also includes the *entitlements* granted to your app. The *certificate* contains the private key you'll use to do the actual signing.

Depending on whether you're registered as an iOS developer, you can use one of the following two ways to obtain a certificate and provisioning profile.

**With an iOS developer account:**

If you have developed and deployed apps iOS using Xcode before, you'll already have your own code signing certificate installed. Use the *security* tool to list your existing signing identities:

~~~
$ security find-identity -p codesigning -v
  1) 61FA3547E0AF42A11E233F6A2B255E6B6AF262CE "iPhone Distribution: Vantage Point Security Pte. Ltd."
  2) 8004380F331DCA22CC1B47FB1A805890AE41C938 "iPhone Developer: Bernhard Müller (RV852WND79)"
~~~

Log into the Apple Developer portal to issue a new App ID, then issue and download the profile [8]. The App ID can be anything - you can use the same App ID for re-signing multiple apps. Make sure you create a *development* profile and not a *distribution* profile, as you'll want to be able to debug the app.

In the examples below I'm using my own signing identity which is associated with my company's development team. I created the app-id "sg.vp.repackaged", as well as a provisioning profile aptly named "AwesomeRepackaging" for this purpose, and ended up with the file AwesomeRepackaging.mobileprovision - exchange this with your own filename in the shell commands below.

**With a regular iTunes account:**

Mercifully, Apple will issue a free development provisioning profile even if you're not a paying developer. You can obtain the profile with Xcode using your regular Apple account - simply build an empty iOS project and extract embedded.mobileprovision from the app container. The NCC blog post explains this process in great detail.

Once you have obtained the provisioning profile, you can check its contents with the *security* tool. Besides the allowed certificates and devices, you'll find the entitlements granted to the app in the profile. You'll need those later for code signing, so extract them to a separate plist file as shown below. It is also worth having a look at the contents of the file to check if everything looks as expected.

~~~
$ security cms -D -i AwesomeRepackaging.mobileprovision > profile.plist
$ /usr/libexec/PlistBuddy -x -c 'Print :Entitlements' profile.plist > entitlements.plist
$ cat entitlements.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>application-identifier</key>
	<string>LRUD9L355Y.sg.vantagepoint.repackage</string>
	<key>com.apple.developer.team-identifier</key>
	<string>LRUD9L355Y</string>
	<key>get-task-allow</key>
	<true/>
	<key>keychain-access-groups</key>
	<array>
		<string>LRUD9L355Y.*</string>
	</array>
</dict>
</plist>
~~~

Note the application identifier, which is a combination of the Team ID (LRUD9L355Y) and Bundle ID (sg.vantagepoint.repackage). This provisioning profile is only valid for the one app with this particular app id. The "get-task-allow" key is also important - when set to "true", other processes, such as the debugging server, are allowed to attach to the app (consequently, this would be set to "false" in a distribution profile).

##### Other Preparations

To make our app load an additional library at startup we need some way of inserting an additional load command into the Mach-O header of the main executable. [Optool](https://github.com/alexzielenski/optool) can be used to automate this process:

~~~
$ git clone https://github.com/alexzielenski/optool.git
$ cd optool/
$ git submodule update --init --recursive
~~~

We'll also use ios-deploy [10], a tools that enables deploying and debugging of iOS apps without using Xcode:

~~~
git clone https://github.com/alexzielenski/optool.git
cd optool/
git submodule update --init --recursive
~~~

To follow the examples below, you also need FridaGadget.dylib:

~~~
$ curl -O https://build.frida.re/frida/ios/lib/FridaGadget.dylib
~~~

Besides the tools listed above, we'll be using standard tools that come with OS X and Xcode (make sure you have the Xcode command line developer tools installed).

##### Patching, Repackaging and Re-Signing

Time to get serious! As you already now, IPA files are actually ZIP archives, so use any zip tool to unpack the archive. Then, copy FridaGadget.dylib into the app directory, and add a load command to the "UnCrackable Level 1" binary using optool.

~~~
$ unzip UnCrackable_Level1.ipa
$ cp FridaGadget.dylib Payload/UnCrackable\ Level\ 1.app/
$ optool install -c load -p "@executable_path/FridaGadget.dylib" -t Payload/UnCrackable\ Level\ 1.app/UnCrackable\ Level\ 1
Found FAT Header
Found thin header...
Found thin header...
Inserting a LC_LOAD_DYLIB command for architecture: arm
Successfully inserted a LC_LOAD_DYLIB command for arm
Inserting a LC_LOAD_DYLIB command for architecture: arm64
Successfully inserted a LC_LOAD_DYLIB command for arm64
Writing executable to Payload/UnCrackable Level 1.app/UnCrackable Level 1...
~~~

Such blatant tampering of course invalidates the code signature of the main executable, so this won't run on a non-jailbroken device. You'll need to replace the provisioning profile and sign both the main executable and FridaGadget.dylib with the certificate listed in the profile.

First, let's add our own provisioning profile to the package:

~~~
$ cp AwesomeRepackaging.mobileprovision Payload/UnCrackable\ Level\ 1.app/embedded.mobileprovision
~~~

Next, we need to make sure that the BundleID in Info.plist matches the one specified in the profile. The reason for this is that the "codesign" tool will read the Bundle ID from Info.plist during signing - a wrong value will lead to an invalid signature.

~~~
$ /usr/libexec/PlistBuddy -c "Set :CFBundleIdentifier sg.vantagepoint.repackage" Payload/UnCrackable\ Level\ 1.app/Info.plist
~~~

Finally, we use the codesign tool to re-sign both binaries:

~~~
$ rm -rf Payload/F/_CodeSignature
$ /usr/bin/codesign --force --sign 8004380F331DCA22CC1B47FB1A805890AE41C938  Payload/UnCrackable\ Level\ 1.app/FridaGadget.dylib
Payload/UnCrackable Level 1.app/FridaGadget.dylib: replacing existing signature
$ /usr/bin/codesign --force --sign 8004380F331DCA22CC1B47FB1A805890AE41C938 --entitlements entitlements.plist Payload/UnCrackable\ Level\ 1.app/UnCrackable\ Level\ 1
Payload/UnCrackable Level 1.app/UnCrackable Level 1: replacing existing signature
~~~

##### Installing and Running the App

Now you should be all set for running the modified app. Deploy and run the app on the device as follows.

~~~
$ ios-deploy --debug --bundle Payload/UnCrackable\ Level\ 1.app/
~~~

If everything went well, the app should launch on the device in debugging mode with lldb attached. Frida should now be able to attach to the app as well. You can verify this with the frida-ps command:

~~~
$ frida-ps -U
PID  Name
---  ------
499  Gadget
~~~

![Frida on non-JB device](Images/Chapters/0x06b/fridaStockiOS.png "Frida on non-JB device")

##### Troubleshooting.

If something goes wrong (which it usually does), mismatches between the provisioning profile and code signing header are the most likely suspect. In that case it is helpful to read the [official documentation](https://developer.apple.com/library/contehttps://support.portswigger.net/customer/portal/articles/1841108-configuring-an-ios-device-to-work-with-burpnt/documentation/IDEs/Conceptual/AppDistributionGuide/MaintainingProfiles/MaintainingProfiles.html "Maintaining Provisioning Profiles") and gaining a deeper understanding of the code signing process. Apple's [entitlement troubleshooting page](https://developer.apple.com/library/content/technotes/tn2415/_index.html "Entitlements Troubleshooting ") is also a useful resource.
