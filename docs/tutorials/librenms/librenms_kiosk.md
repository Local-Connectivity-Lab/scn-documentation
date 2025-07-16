---
title: LibreNMS Kiosk
---

#LibreNMS Kiosk

We have a TV in the scn/hack space and you can use it to show LibreNMS

## Usage
Turn on the TV with the roku remote, It may be automatically set up to display the HDMI port. Make sure the RPI is plugged into the TV and the HDMI port of the RPI. Also make sure the RPI is plugged into power. If the RPI was rebooted, it will go into the terminal. Log into the RPI using the username and password in vaultwarden. Once into the terminal, run the command `startx`. It will open up LibreNMS (backed by a chromium browser). Log in using preferably a readonly user like `librenms_viewer` the password is in vaultwarden. You can now simply turn off the computer and when you turn it on again it will come up.


## How it was developed
Flash a minimal version of raspbian on an rpi, install xinit and chromium. create an .xinitrc file and put the following in it

```
xset s off
xset -dpms
xset s noblank
exec /usr/bin/chromium-browser --no-sandbox --disable-gpu --incognito --kiosk --window-size=1920,1080 librenms.seattlecommunitynetwork.org
```