# rsyslog-docs
Some notes on getting rsyslog working

This document is intended for anyone installing and configuring rsyslog on Red Hat 7.0. 
It is not intended to be fully comprehensive but it does cover the gotchas that I encountered,
and hopefully it will be able to prevent you from doing the same.

## Installation
rsyslog comes installed by default on Red Hat 7.0 machines. However the version that you get
is positively ancient and it is worth while to install the newest version from rsyslog.
The rsyslog developers have taken great care to make sure that installation is a breeze and 
if you follow the instructions on the rsyslog website then there should be no problem. The 
steps are basically:
1. Create a rsyslog repository.
1. Run yum install rsyslog

This should install the latest available rsyslog for you which at the time of writing is 8.11.

The first gotcha is that rsyslog come with problematic config files, regardless of the version
you install. So the first thing to get rid of is the config in /etc/rsyslog.d/listen.conf. I 
commented it out, if you don't and you are reading your logs /var/log/messages then you will see
an error message whenever you start rsyslog.

Another important point to remember is that rsyslog can be configured in two different ways. 
There is an old style method and a new style method. The two do not mix well and you will 
encounter a world of pain if you do so. 

## Aims
My aim was to achieve reliable and secure logging between a client machine and a centralised master server.
I wanted the setup to be robust enough to handle the master server being down 

## Config
### Client config
```
#### MODULES ####
module(load="imuxsock") # provides support for local system logging (e.g. via logger command)
module(load="imklog")   # provides kernel logging support (previously done by rklogd)
module(load="omrelp")
module(load="impstats" interval="10" severity="7")

if $syslogtag contains 'rsyslogd-pstats' then {
     action(type="omfile" queue.type="linkedlist" queue.discardmark="980"
            name="pstats" file="/var/log/pstats")
     stop
}

#### GLOBAL DIRECTIVES ####
$IncludeConfig /etc/rsyslog.d/*.conf

#### RULES ####
#*.info;mail.none;authpriv.none;cron.none                /var/log/messages
local3.*                                                /var/log/testing
#authpriv.*                                              /var/log/secure
#mail.*                                                  /var/log/maillog
#cron.*                                                  /var/log/cron
#*.emerg                                                 :omusrmsg:*
#uucp,news.crit                                          /var/log/spooler
#local7.*                                                /var/log/boot.log

# ### begin forwarding rule ###
action( type="omrelp"
        name="logtoserver"
        target="192.168.8.134"
        port="514"
        queue.size="5000"
        queue.type="LinkedList"
        queue.spoolDirectory="/var/lib/rsyslog"
        queue.filename="testforwardingqueue"
        queue.lowwatermark="2000"
        queue.highwatermark="3500"
        queue.discardmark="5000"
        queue.maxfilesize="1g"
        queue.saveonshutdown="on"
        action.ResumeInterval="10"
        action.ResumeRetryCount="-1"
        action.reportSuspension="on"
        action.reportSuspensionContinuation="on"
      )
```

### Server config
## Debugging



