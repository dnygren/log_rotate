# log_rotate
Rotate and compress log files to avoid running out of storage. 

Perform hourly checks of log sizes, hourly rotation if a certain size exceeded, and
daily rotation if not. Do not exceed 28 copies of any log file.

<pre>
nygren@server-1:/var/log$ cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
</pre>
(No change by me. Just shows Debian hourly cron jobs run at 17 minutes past the hour)
<pre>
nygren@server-1:/etc/cron.daily$ ls -la
total 79
drwxr-xr-x   2 root root   14 May  9 09:52 .
drwxr-xr-x 128 root root  251 May 17 06:04 ..
-rwxr-xr-x   1 root root  539 Sep 19  2017 apache2
-rwxr-xr-x   1 root root 1474 Sep 13  2017 apt-compat
-rwxr-xr-x   1 root root  314 Apr 18  2017 aptitude
-rwxr-xr-x   1 root root  355 Oct 25  2016 bsdmainutils
-rwxr-xr-x   1 root root  384 Dec 12  2012 cracklib-runtime
-rwxr-xr-x   1 root root 1597 May  2  2016 dpkg
-rwxr-xr-x   1 root root   89 May  5  2015 logrotate
-rwxr-xr-x   1 root root 1065 Dec 13  2016 man-db
-rwxr-xr-x   1 root root 1386 Aug  8  2017 ntp
-rwxr-xr-x   1 root root  249 Feb 24  2017 passwd
-rw-r--r--   1 root root  102 May  3  2015 .placeholder
-rwxr-xr-x   1 root root  441 May 25  2017 sysstat

nygren@server-1:/etc/cron.daily$ cat logrotate
#!/bin/sh

test -x /usr/sbin/logrotate || exit 0
/usr/sbin/logrotate /etc/logrotate.conf

nygren@server-1:/etc/cron.daily$ cd ..
nygren@server-1:/etc$ cd cron.hourly
nygren@server-1:/etc/cron.hourly$ ls -la
total 22
drwxr-xr-x   2 root root   3 May  9 09:52 .
drwxr-xr-x 128 root root 251 May 17 06:04 ..
-rw-r--r--   1 root root 102 May  3  2015 .placeholder

</pre>
( Move logrotate from /etc/cron.daily to /etc/cron.hourly )
<pre>
nygren@server-1:/etc/cron.hourly$ cd ..
nygren@server-1:/etc$ cd cron.daily
nygren@server-1:/etc/cron.daily$ sudo mv logrotate ../cron.hourly/logrotate
[sudo] password for nygren:

nygren@server-1:/etc$ cd ../cron.hourly
nygren@server-1:/etc/cron.hourly$ ls -la
total 26
drwxr-xr-x   2 root root   4 May 17 17:22 .
drwxr-xr-x 128 root root 251 May 17 17:22 ..
-rwxr-xr-x   1 root root  89 May  5  2015 logrotate
-rw-r--r--   1 root root 102 May  3  2015 .placeholder
</pre>
( No changes to /etc/logrotate.conf , but shown here as it calls files in /etc/logrotate.d)
<pre>
nygren@server-1:/etc$ cat logrotate.conf
# see "man logrotate" for details
# rotate log files weekly
weekly

# keep 4 weeks worth of backlogs
rotate 4

# create new (empty) log files after rotating old ones
create

# uncomment this if you want your log files compressed
#compress

# packages drop log rotation information into this directory
include /etc/logrotate.d

# no packages own wtmp, or btmp -- we'll rotate them here
/var/log/wtmp {
    missingok
    monthly
    create 0664 root utmp
    rotate 1
}

/var/log/btmp {
    missingok
    monthly
    create 0660 root utmp
    rotate 1
}

# system-specific logs may be configured here
</pre>

( Changed /etc/logrotate.d/rsyslog to rotate at size of 10MB when checked hourly or daily )

<pre>
# ###
# Note that /var/log/lastlog isn't included in the rotation scheme as it is
# a sparse file that really isn't as large as it appears in  "ls".
# $ ls -lh lastlog
# -rw-rw-r-- 1 root utmp 12M May 16 15:15 lastlog
# $ du -h lastlog
# 521K    lastlog
#
# ###
# Commented out separate syslog entry and moved it with the others.
#/var/log/syslog
#{
#       rotate 7
#       daily
#       missingok
#       notifempty
#       delaycompress
#       compress
#       postrotate
#               invoke-rc.d rsyslog rotate > /dev/null
#       endscript
#}
#
# Added /var/log/syslog to the below list after commenting out the separate
# entry above.
/var/log/syslog
/var/log/mail.info
/var/log/mail.warn
/var/log/mail.err
/var/log/mail.log
/var/log/daemon.log
/var/log/kern.log
/var/log/auth.log
/var/log/user.log
/var/log/lpr.log
/var/log/cron.log
/var/log/debug
/var/log/messages
{
#       rotate 4
#       weekly
#
# ###
# Keep 28 logs instead of 4 weekly copies of the logs. This is likely to be 28
# daily copies, but if maxsize is exceeded, it could be 28 hourly copies if
# logrotate is called hourly.  This number of copies will give us coverage over
# 24 hours so if something is going wrong there is an opportunity to see a log
# file at the start, and also cover both sides of the planned two week MMO
# power cycle. 
        rotate 28
        daily
# Rotate now if maxsize exceeded
        maxsize 10M
        missingok
        notifempty
        compress
# Do not delay compression of a rotated log, as they can get big really fast.
#       delaycompress
        sharedscripts
        postrotate
                invoke-rc.d rsyslog rotate > /dev/null
        endscript
}
</pre>

