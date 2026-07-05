+++
title = 'Linux cron jobs give you the power'
date = 2023-06-09T11:00:00-07:00
draft = false
tags = ['linux']
+++


As an Asian, I'm familiar with my parents always keeping a close eye on me, checking if I'm studying, gaming, or chatting. It's like they have a background process running, giving those bombastic side eye 👀 glances. Similarly, Linux's cron job operates silently in the background, automating tasks. I found this parallel intriguing: both involve monitoring and control. There is a cron daemon already inside Linux that helps you execute commands or scripts at predefined time intervals. We can automate tasks like

1.  Periodically backing up the data
    
2.  Periodically deleting unused files or folders
    
3.  Periodically Scrape some website periodically
    
4.  Logging program data and storing
    
5.  Send yourself a motivation quotes
    
6.  Periodically change the wallpapers
    

![drink water](https://gifdb.com/images/high/daily-reminder-to-drink-water-funny-meme-orvd01mhg5mtgvza.gif)

or create yourself a reminder which we will create btw. *(Periodically counter - 5)*

The list can go on; there are endless possibilities for what we can do with a cron job. Basically, cron is a daemon service that constantly checks the system time against the schedule defined in the cron table. When the current time matches the schedule, `cron` triggers the associated commands. It's a powerful tool for system administrators.

### Running Cron job

cron is already installed in `apt-based` Linux, but if it not installed in your system there are many tutorials on how to install it. Mine is arch Linux, to install **crontab** (command line utility for cron alias for cron table).

```bash
pacman -Syu cronie
systemctl enable --now cronie.service
```

systemctl is another command line utility that manages systemd and service, you can even manage your own service start it on boot and run as a background process.

1.  Add new crontab entry : `crontab -e` (choose any editor)
    
2.  Head over to `/var/spool/cron` there is a file with your cron jobs under your username
    
3.  open that file in nvim or any other preferred editor
    

in that file we can set up the cron jobs, it accepts five different time fields which is mentioned below, to play around with time and schedules of cron you can check this [link](https://crontab.guru/).

```bash
* * * * * <commands to execute>
| | | | |
| | | | +---- Day of the week (0 - 6) (Sunday to Saturday, 0 and 7 both represent Sunday)
| | | +------ Month (1 - 12 or Jan - Dec)
| | +-------- Day of the month (1 - 31)
| +---------- Hour (0 - 23)
+------------ Minute (0 - 59)
```

you can directly start writing \* \* \* \* \* echo "hello world" in your terminal. To understand how it works lets give some cj.

### echo out periodically in txt file

Inside the file `/var/spool/cron` write `* * * * * echo "Hello world!" >> ~/test.txt` on the top line. you can add other crons below it. We can check if the cron is runnign or not by command `crontab -l`, We can also view if process is running by command `pgrep crond`, it shows the process id of the cron. Looks like, we are running cron job, lets view out file in ~/test.txt `cat ~/test.txt`, it has hello world in it. Hell Yeah!, we just created a cron job.

### notification service

As mentioned earlier we are going to make a reminder to drink water every hour. Inside the file `/var/spool/cron` write `* 1 * * * ~/`[`script.sh`](http://script.sh) I have written a simple script to notify me to drink the water:

```plaintext
export DISPLAY=:0
export XDG_RUNTIME_DIR=/run/user/$(id -u)
/usr/bin/notify-send "Drink some water dude! 🌊"
```

### monitoring tool

Hmm, I recently made an API that sends images or documents to a Telegram bot. I created it so I can send my files via the terminal, [spymyson](https://spymyson.vercel.app/). It's quite fun. I also thought about monitoring my activity throughout the day, so I wrote a simple script to send me screenshots periodically. Inside the file `/var/spool/cron`, write `0 9-17 * * 0-5 ~/`[`script.sh`](http://script.sh). It runs from 9 AM to 5 PM every Sunday to Friday.

```bash
#!/bin/bash
export DISPLAY=:1
export XDG_RUNTIME_DIR=/run/user/$(id -u)

API_ENDPOINT="https://spymyson.vercel.app/api/send"
AUTH_TOKEN="TOKEN GIVEN BY BOT"
DIR="$HOME/Pictures/Screenshots"
FILENAME="screenshot.png"
FILEPATH="$DIR/$FILENAME"

mkdir -p $DIR

/usr/bin/scrot -F $FILEPATH -p -q 100

curl -X POST -F "file=@$FILEPATH" -H "Authorization: $AUTH_TOKEN" $API_ENDPOINT

rm $FILEPATH
```

Yeah, that's it. It's a fairly simple thing to do, but it's also powerful. Cron jobs are really fun :)
