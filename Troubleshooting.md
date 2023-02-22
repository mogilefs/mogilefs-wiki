# Where To Look #

If something is going wrong, the first place to look is your app. Are you handling client errors correctly? Are you reporting/logging them anywhere?

The next place are your trackers. If you have **syslog** configured correctly, they should log various things.

If not, or if you want to spot check a particular tracker, wield the power of telnet:
```
telnet tracker 7001
!watch
(enjoy)
```

If you restart the tracker with 'debug = 1' or 'debug = 2', you will get more log messages.
You can put this setting in your mogilefsd.conf file.

