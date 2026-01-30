I'll help you set up a launchd job (the macOS-preferred way over cron) to run SelfControl every morning at 6 AM for 3 hours.

First, let me check if you have SelfControl installed and find its CLI path.

Bash Check if SelfControl is installed and find CLI
IN
ls -la /Applications/SelfControl.app/Contents/MacOS/ 2>/dev/null || echo "SelfControl not found in /Applications"

OUT
total 13664
drwxr-xr-x@  6 vivian  admin      192 Nov  1  2021 .
drwxr-xr-x@ 10 vivian  admin      320 Nov  1  2021 ..
-rwxr-xr-x@  1 vivian  admin  3010144 Nov  1  2021 SCKillerHelper
-rwxr-xr-x@  1 vivian  admin   812336 Nov  1  2021 SelfControl
lrwxr-xr-x   1 vivian  admin       17 Oct 19  2024 org.eyebeam.SelfControl -> ./selfcontrol-cli
-rwxr-xr-x@  1 vivian  admin  3168688 Nov  1  2021 selfcontrol-cli
SelfControl is installed. Let me check its CLI usage.

Bash Check selfcontrol-cli usage
IN
/Applications/SelfControl.app/Contents/MacOS/selfcontrol-cli --help 2>&1 || /Applications/SelfControl.app/Contents/MacOS/selfcontrol-cli 2>&1 | head -30

OUT
SelfControl CLI Tool v4.0.2
Usage: selfcontrol-cli [--uid <controlling user ID>] <command> [<args>]

Valid commands:

    start --> starts a SelfControl block
        --blocklist <path to saved blocklist file>
        --enddate <specified end date for block in ISO8601 format>
        --settings <other block settings in JSON format>

    is-running --> prints YES if a SelfControl block is currently running, or NO otherwise

    print-settings --> prints the SelfControl settings being used for the active block (for debug purposes)

    version --> prints the version of the SelfControl CLI tool

--uid argument MUST be specified and set to the controlling user ID if selfcontrol-cli is being run as root. Otherwise, it does not need to be set.

Example start command: selfcontrol-cli start --blocklist /path/to/blocklist.selfcontrol --enddate 2021-02-12T06:53:00Z
Let me check if you have an existing blocklist saved.

Bash Check existing SelfControl preferences
IN
ls -la ~/Library/Preferences/org.eyebeam.SelfControl.plist 2>/dev/null && defaults read org.eyebeam.SelfControl 2>/dev/null | head -50

OUT
-rw-------@ 1 vivian  staff  303 Jan 29 21:31 /Users/vivian/Library/Preferences/org.eyebeam.SelfControl.plist
{
    BlockAsWhitelist = 0;
    BlockDuration = 540;
    Blocklist =     (
        "www.youtube.com"
    );
    FirstBlockStarted = 1;
    GetStartedShown = 1;
    SUHasLaunchedBefore = 1;
    SULastCheckTime = "2026-01-30 02:31:07 +0000";
    SULastProfileSubmissionDate = "2026-01-25 11:26:43 +0000";
    SuppressLongBlockWarning = 1;
}
Bash Check for exported blocklists
IN
ls -la ~/Documents/*.selfcontrol 2>/dev/null || echo "No exported blocklists found"

OUT
No exported blocklists found
(eval):1: no matches found: /Users/vivian/Documents/*.selfcontrol
You have SelfControl configured with YouTube in your blocklist. Let me set up the automated schedule.

I'll create:

An exported blocklist file
A shell script to start SelfControl
A launchd plist to run at 6 AM daily
Bash Create directory for SelfControl config
IN
mkdir -p ~/.selfcontrol

OUT
0
Write blocklist.selfcontrol
8 lines
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<array>
	<string>www.youtube.com</string>
</array>
</plist>
Write start-block.sh
24 lines
#!/bin/bash
# SelfControl morning block script
# Starts a 3-hour block (6 AM - 9 AM)

BLOCKLIST="$HOME/.selfcontrol/blocklist.selfcontrol"
CLI="/Applications/SelfControl.app/Contents/MacOS/selfcontrol-cli"
UID_NUM=$(id -u)

# Calculate end time (3 hours from now)
END_DATE=$(date -v+3H -u +"%Y-%m-%dT%H:%M:%SZ")

# Check if block is already running
if [ "$($CLI is-running)" = "YES" ]; then
    echo "$(date): SelfControl block already running, skipping."
    exit 0
fi

echo "$(date): Starting SelfControl block until $END_DATE"

# Start the block (requires root)
sudo "$CLI" --uid "$UID_NUM" start --blocklist "$BLOCKLIST" --enddate "$END_DATE"

echo "$(date): SelfControl block started successfully"
Bash Make script executable
IN
chmod +x ~/.selfcontrol/start-block.sh

OUT
0
Write com.vivian.selfcontrol.plist
27 lines
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.vivian.selfcontrol</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>/Users/vivian/.selfcontrol/start-block.sh</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Hour</key>
        <integer>6</integer>
        <key>Minute</key>
        <integer>0</integer>
    </dict>
    <key>StandardOutPath</key>
    <string>/Users/vivian/.selfcontrol/selfcontrol.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/vivian/.selfcontrol/selfcontrol.log</string>
    <key>RunAtLoad</key>
    <false/>
</dict>
</plist>
Now I need to set up passwordless sudo for the selfcontrol-cli command (since it runs unattended). This requires creating a sudoers rule.

Write selfcontrol-sudoers
3 lines
# Allow vivian to run selfcontrol-cli without password
vivian ALL=(ALL) NOPASSWD: /Applications/SelfControl.app/Contents/MacOS/selfcontrol-cli
Here's what I've created:

Files created:

~/.selfcontrol/blocklist.selfcontrol - Your blocklist (currently has youtube.com)
~/.selfcontrol/start-block.sh - Script to start a 3-hour block
~/Library/LaunchAgents/com.vivian.selfcontrol.plist - Runs daily at 6 AM
To complete setup, run these commands:


# 1. Set up passwordless sudo for selfcontrol-cli
sudo cp ~/.selfcontrol/selfcontrol-sudoers /etc/sudoers.d/selfcontrol
sudo chmod 440 /etc/sudoers.d/selfcontrol

# 2. Load the launchd job
launchctl load ~/Library/LaunchAgents/com.vivian.selfcontrol.plist
To test it now:


~/.selfcontrol/start-block.sh
To add more sites to block, edit ~/.selfcontrol/blocklist.selfcontrol and add more <string>site.com</string> entries.

Useful commands:

Check if scheduled: launchctl list | grep selfcontrol
View logs: cat ~/.selfcontrol/selfcontrol.log
Unload: launchctl unload ~/Library/LaunchAgents/com.vivian.selfcontrol.plist