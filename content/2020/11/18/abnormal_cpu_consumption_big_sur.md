Abnormal CPU Usage After Upgrading to Big Sur(11.0.1)
===

- date: 2020-11-30
- tags: Big Sur, OSX

------------

# Background

I upgraded to Big Sur several days ago. There were still some annoying issues under Catalina after several releases. For instance the `Mail.app` may occupy over 200% CPU and its memory consumption keep raising. I have seen it consumed about 35GB of memory. And the conditions causing it are unknown. I have tried even reimport all the mails. To take a rest from keeping attention on its memory consumption I wrote a cron job to do monitoring. It ran every 5 minutes and would kill it if it consumed more than 2 GB memory. When I saw the dot of Mail.app in the dock disappeared I knew that it was killed on the way of out of control. It's a tricky solution.

So when I saw the new release called Big Sur I thought of the risk for a while. I really wanted my Mail.app issue solved but no more nasty bugs. Finally I did the upgrading.

# After Upgrading

Upgrading process cost me about an hour and be honestly I don't like the new style of UI which makes me feeling like my 2015mid MBP turns to be an iPad pro.

But anyway it's amazing the Mail.app issue seems disappeared after recreating all my mail accounts(There was still a long way before this). But something was wrong that the fan just didn't stop for a while, kept working hard. I opened the monitor and found `secd` process consume 100% CPU.

Life is too hard for me.

Details can be reached [here](https://discussions.apple.com/thread/252054223?login=true).

# Temporary Workaround

Instead trying to relogin my iCloud account I made a temporary workaround. I wrote another cron job to kill `cloudd` periodically which makes it easier waiting a new release of Big Sur.

My script is:

```shell
#!/bin/sh

PID=$(pgrep -u $UID ^cloudd$)
if [ "no$PID" = "no" ]; then
	exit
fi
kill $PID

sleep 30
PID=$(pgrep -u $UID ^cloudd$)
if [ "no$PID" = "no" ]; then
        exit
fi
kill $PID
```

Save it to your home directory (eg. `~/check_cloudd`) and grant it executable by

```shell
chmod +x ~/check_cloudd
```

You can add it into your cron by

```shell
crontab -e
```

and take this for reference (use your own path)

```
* * * * * ~/check_cloudd
```

It will kill `cloudd` every 30 seconds.

# Final Shot

The temporary solution worked for about ten days. But in a morning it doesn't work anymore. When `cloudd` restarted it ran into heavy load immediately. And there still haven't existed a new release yet.

So I gave up and logged out my `Apple ID` finally. Restarted OSX, waited it be stable, relogged in my Apple ID and clean all duplicated copy of data like iCloud Drive, etc. .

Everything seems back to normal which should be verified by days in future.


