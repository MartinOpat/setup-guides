# Motion
A self-hosted camera security system using [Motion](https://motion-project.github.io/).

Once again, the main assumption is you are on a debian/ubuntu-based system.

## Basic setup
Install motion and dependencies:
```bash
https://motion-project.github.io/
```

Edit the config:
```
sudo nano /etc/motion/motion.conf
```

You can set these to your preferences, but most important settings are:
- framerate 15  # I recommend 10-15
- threshold 1500  # Number of pixels that trigger motion should be relatively large to avoid triggering camera for every speck of dust.
- minimum_motion_frames 161  # N.o. of images that must contain motion to trigger an event - empirically tuned (for my camera tho)
- event_gap 60  # How long (# frames) of no event (i.e no motion) before recording stops.
- pre_capture 15  # For a security camera, we want a good amount of frames before motion begins.
- post_capture 0  # But we do not need to capture frames after motion stops, but of course, up to your own preferences.

After that, you can (re)start motion via:
```bash
sudo systemctl restart motion
```

You should be able to see your camera on `http://localhost:8080` (ui) or `http://localhost:8081` (just camera) (or whatever port you used in the config file).
Note: If you used the [language tool setup](../language-tool/language-tool.md) you might wanna change the default port of one of these, as they both use 8080/8081.

## Automatic email notification on movement detection
In the `/etc/motion/motion.conf` file, set:
```ini
# Command to be executed when an event starts.
on_event_start /etc/motion/send_warning_email.sh
```

You need a Google App Password (normal password will NOT work). Go to your Google Account > Security > 2-Step Verification > App Passwords. Create one named, e.g.: "UbuntuCamera".

Creat the `/etc/motion/send_warning_email.sh` bash script file and put the following code in:
```bash
#!/bin/bash
sendemail -f your_email@gmail.com -t your_email@gmail.com -u "Security Alert" -m "Movement detected! Recording started." -s smtp.gmail.com:587 -o tls=yes -xu your_email@gmail.com -xp "your_app_password" >> /tmp/motion_email.log 2>&1
```

Restart motion for it to take effect: `sudo systemctl restart motion`.
