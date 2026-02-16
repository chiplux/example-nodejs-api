# Cronjob to Prune Unused Docker Images and Docker Build Cache Every Hour

## Prerequisites
- Docker
- Cron
- Root Access

## Step 1 — Verify docker path (IMPORTANT)

Command:
```
which docker
```
Result:
```
/usr/bin/docker
```

## Step 2 — Create Prune Script
Create a small script so cron stays clean.
```
sudo nano /usr/local/bin/docker-prune-hourly.sh
```

Content of the script:
```
#!/usr/bin/env bash
set -e

/usr/bin/docker image prune -af
/usr/bin/docker builder prune -af
```

Make the script executable:
```
sudo chmod +x /usr/local/bin/docker-prune-hourly.sh
```

## Step 3 — Add Cron Job to Prune Unused Docker Images and Docker Build Cache Every Hour

Command:
```
crontab -e
```
Add this line:
```
0 * * * * /usr/local/bin/docker-prune-hourly.sh >> /var/log/docker-prune.log 2>&1
```

## Step 4 — Test Manually
Test the script manually to make sure it works:
```
sudo /usr/local/bin/docker-prune-hourly.sh
```

## Step 5 — Check Log
Example log result:
```
ubuntu@ip-10-0-6-115:~$ sudo crontab -l
# Edit this file to introduce tasks to be run by cron.
#
# Each task to run has to be defined through a single line
# indicating with different fields when the task will be run
# and what command to run for the task
#
# To define the time you can provide concrete values for
# minute (m), hour (h), day of month (dom), month (mon),
# and day of week (dow) or use '*' in these fields (for 'any').
#
# Notice that tasks will be started based on the cron's system
# daemon's notion of time and timezones.
#
# Output of the crontab jobs (including errors) is sent through
# email to the user the crontab file belongs to (unless redirected).
#
# For example, you can run a backup of all your user accounts
# at 5 a.m every week with:
# 0 5 * * 1 tar -zcf /var/backups/home.tgz /home/
#
# For more information see the manual pages of crontab(5) and cron(8)
#
# m h  dom mon dow   command
0 * * * * /usr/local/bin/docker-prune-hourly.sh >> /var/log/docker-prune.log 2>&1
ubuntu@ip-10-0-6-115:~$ sudo cat /var/log/docker-prune.log
Total reclaimed space: 0B
Total:	0B
Total reclaimed space: 0B
Total:	0B
Total reclaimed space: 0B
Total:	0B
```

