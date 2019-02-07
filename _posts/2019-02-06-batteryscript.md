---
layout: post
title: Custom Mac OS Low-Battery Notification using AppleScript
comments: false
---

For some time, I was bothered by the fact that I would not receive a low battery warning on my MacBook until the critical 5%. In a panic, I'd have to search furiously for A) my charger and B) an outlet. I was also surprised to find out that there wasn't a native way to change the options related to the warning, especially since other Apple products would give warnings much earlier than 5%. 

Just over a year ago, I decided to look into creating my own notification script to solve this small annoyance and to alert me at a more reasonable percentage. I was actually able to get something working with the help of this link from lifehacker [here](https://www.lifehacker.com.au/2011/02/give-your-mac-a-more-attention-grabbing-low-battery-warning/).  

This sadly only lasted until the recent Mojave update when things just stopped. I first realized this while sitting in a cafe, multitasking away with my usual 23 tabs open and far from any discernible outlets. I decided to take another look at the problem and find out what was going on.


If you're in a hurry, you can skip to the [solution](#solution), but if you have some time, you may enjoy the read.

**Note: This post will mostly cover the details of writing this battery notification from scratch. If you're looking to fix you're previously working script just as I was, please see the [notes](#notes) section near the end for some tips that helped me out.**

## Why AppleScript?

There is no *real* reason to use AppleScript over something else here. AppleScript is generally a more user-friendly and accessible scripting language and I was stubborn on finding a fix to my issue that was already written in the language.

A year ago I was new to automation tasks, especially on Mac, and following the methods in the original article, I had thought that AppleScript was just the way to do these sorts of things. This is certainly not the case and while delving into this problem, I actually experimented with a few alternative approaches, including using a bash script, a tutorial on which will be available shortly.

You'll see that many of the lines in the completed AppleScript code actually use quite a bit of shell scripting and that the completed Bash version also calls on some AppleScript functionality (namely `osascript`). So, it really is dealer's choice here.


## Launchd vs. Cron

Cron is the traditional Linux "go-to" when it comes to script automation and scheduling. Although previously supported in Mac OS, it is now deprecated, with functionality being "absorbed" into Apple's launchd. You could still use either in this case; I've chosen Launchd for the same reason I chose AppleScript for this project.

## The Solution <a name="solution"></a>


I've made a few tweaks to the original AppleScript file that I much prefer; the original script is much more obnoxious and turns the volume to 100% and continually shouts at you to find an outlet.

Throughout this process, I've made a couple variations but the one I've settled on, will run in the background to check if your battery levels are below 20% and remind you every 10 minutes to plug in until you reach below 10%, in which case, it will remind you every 5 minutes to find and outlet. (This script can be found in my [Github](https://github.com/jmmoloney/batteryscript/tree/master/applescript) with `...keepRunning...` in the filename.)

There shouldn't be too much battery drain from this running process, but if I do notice anything, I may revert to a less frequent check than this one. See the [notes](#notes) for tips on doing that.

### Step One: Write your script
Script Editor is an application that should already exist on your Mac. Open that up and copy and paste the following code into the editor and make any adjustments you'd like. I'd first save it as a text file so that it's easier to edit later if need be.

```applescript
set Cap to (do shell script "ioreg -w0 -l | grep ExternalChargeCapable")tell Cap to set {wallPower} to {last word of paragraph 1}if wallPower = "Yes" then	return 0else	set Pct to (do shell script "pmset -g batt | grep -Eo \"\\d+%\" | cut -d% -f1")	if Pct > 10 and Pct ≤ 20 then		display notification "Less than 20% Battery Remaining, plug in soon." with title "Low Battery" sound name "Basso"
		delay 600	else if Pct ≤ 10 then		display notification "Less than 10% Battery, plug in now." with title "Critical Battery" sound name "Sosumi"
		delay 300	end ifend if

```
### Step Two: Export and save
Export your applescript as a Script (`.scpt`) file named `batteryScript.scpt` and save in your `/etc/` folder. It should already be executable, but if not, run the following in your terminal:

```bash
chmod +x /path/to/file/batteryScript.scpt
```

**Note:** you can also save the file as a bundle or app, this increases the size (slightly), may require some changes in the plist file (later) and apps generally take a bit longer to load and run.

### Step Three: Write your .plist file
A `.plist` file is a 'property list' file that contains the information about you daemon or agent in order to process the requests. The variables set here actually help you launch your agent and get your process running.

Copy and paste the following into your favourite text editor and save as `batteryAlert.plist`. Then copy the file into your `~/Library/LaunchAgents` directory.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>KeepAlive</key>
  <true/>
  <key>Label</key>
  <string>batteryAlert</string>
  <key>LowPriorityIO</key>
  <true/>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/bin/osascript</string>
    <string>/etc/batteryScript.scpt</string>
  </array>
  <key>RunAtLoad</key>
  <true/>
  <key>ServiceDescription</key>
  <string>Battery Alert</string>
</dict>
</plist>
```
Depending on whether you want your script to be available to all users of your computer, or limited to just your profile, you will have to save the `.plist` file in one of two places. Briefly, if you want your script to run as root (for all users) you should place this file in `/Library/LaunchDaemons` instead.

### Step Four: Load your script
In order to load your script (and get it to start running), run the following command in your terminal.

```bash
launchctl load ~/Library/LaunchAgents/batteryAlert.plist
```

Your process should now be ready to go. 

We can check that all is well with the following command.

```bash
launchctl list | grep batteryAlert 
```
We should see an exit code of 0 in the second column, as well as a process ID in the first column if the process is currently running. 
Don't panic if there is no ID just yet, it is more important that the exit code is 0 and will receive an ID when it is actually running. 

<p align = "center">
	<a href ="https://raw.githubusercontent.com/jmmoloney/batteryscript/master/screenshots/batteryScript_loadSuccess.png" target="_blank">
		<img src="https://raw.githubusercontent.com/jmmoloney/batteryscript/master/screenshots/batteryScript_loadSuccess.png"/>
	</a>
</p>      
If you ever need to unload or reload your `.plist`, run the following and then run the load command again. This prevents you from having to restart to see changes (which many tutorials may suggest).

```bash
launchctl unload ~/Library/LaunchAgents/batteryAlert.plist`
```
 

## The Result

Well that's it! Your script should be up and running and display low-battery notifications as shown below.

If you're anything like me, the time can get away from you, and before you know it, you're at critically low battery levels. Hopefully, this script will help you out a bit with that.

The files used in this post are also available on my [GitHub](https://github.com/jmmoloney/batteryscript).

<p align="center">
	<a href= "https://raw.githubusercontent.com/jmmoloney/batteryscript/master/screenshots/20_batteryWarning.png" target="_blank">
		<img src="https://raw.githubusercontent.com/jmmoloney/batteryscript/master/screenshots/20_batteryWarning.png"/>
	</a>
</p>
<p align="center">
	<a href= "https://raw.githubusercontent.com/jmmoloney/batteryscript/master/screenshots/10_batteryWarning.png" target="_blank">
		<img src="https://raw.githubusercontent.com/jmmoloney/batteryscript/master/screenshots/10_batteryWarning.png"/>
	</a>
</p>

___
___

## Notes 

A few configurable things to note:

1. The `KeepAlive` variable will keep the script running. Change this to false if you do not want this functionality.
2. In order to run this script periodically instead, remove the `delay n` lines from your `.scpt` file and add the following as the last element in the `dict` in your `.plist` file.

   ```applescript
     <key>StartInterval</key>
     <integer>600</integer>
   ```
I've already written some of the variations you could do. They're all available on my Github, linked below.
<a name="notes"></a>

**If you've followed the instructions of the original article posted above, there are two main things to change in order to get your script running.**

1. The line to query the battery percentage no longer works in Mojave. To remedy this, I completely changed the line following the `else` in the `batteryScript` file to the following:

   ```applescript
   set Pct to (do shell script "pmset -g batt | grep -Eo \"\\d+%\" | cut -d% -f1")
```
2. Simply making the `.applescript` file executable within my `/etc/` folder didn't actually make my script work. I needed to export the compiled script from ScriptEditor as a Script (`.scpt`) (Or as I later learned, as a ScriptBundle(`.scptd`) or Application(`.app`) works too). This may have been obvious to some, but was certainly not clear to me.

These two things, *should* allow for your script to run just fine.

___

## Additional Resources

* [My Github BatteryScript Repo](https://github.com/jmmoloney/batteryscript)
* [Original Battery Notification How-To](https://www.lifehacker.com.au/2011/02/give-your-mac-a-more-attention-grabbing-low-battery-warning/)
* [Unofficial Launchd Guide](http://www.launchd.info/)
* [Official Launchd Documentation](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html)