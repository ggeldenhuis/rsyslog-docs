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
local3.*                                                 /var/log/testing
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
```
module(load="imuxsock") # provides support for local system logging (e.g. via logger command)
module(load="imklog")   # provides kernel logging support (previously done by rklogd)
module(load="imrelp")

main_queue(
  queue.size="1000000"   # capacity of the main queue
  queue.debatchsize="1000"  # process messages in batches of 1000 and move them to the action queues
  queue.workerthreads="4"  # 2 threads for the main queue
)

input(type="imrelp" port="514")
$ActionFileDefaultTemplate RSYSLOG_TraditionalFileFormat
$IncludeConfig /etc/rsyslog.d/*.conf

local3.*                                                /var/log/testing
```

## Testing
To generate logs I used a very simple for loop with a unique sequence. This meant that I can easily grep 
my logs for a particular sequence and then count the messages to check if it all arrived.

```
# On local machine
for i in $(seq 1 6000); do logger -p local3.error "############### TESTING ########### SEQ001-${i}"; done

# On remote machine
grep 'SEQ001' /var/log/testing -c
```

I tried a number of scenarious to break the connection which produced different results and highlighted
config issues I had.
* Enable firewall by running ```systemctl start firewalld```. I were using the default firewall config that comes with Red Hat which only allows SSH connections.
* Disable network by running ```systemctl stop network``` on the rsyslog server
* Disable network by disconnecting the network in VMWare.


## Debugging
There is a number of ways to do debugging in rsyslog and depending on how deep seated your problem is you should choose
what is most appropriate.
My first personal port of call was to enable some additional logging within the config:
```
module(load="impstats" interval="10" severity="7")

if $syslogtag contains 'rsyslogd-pstats' then {
     action(type="omfile" queue.type="linkedlist" queue.discardmark="980"
            name="pstats" file="/var/log/pstats")
     stop
}
```

This will load the impstats module and the second piece of code will conditionally
log pstats tagged messages to /var/log/pstats
You can then tailf this file to see what is happening. This is usefull but you will soon
realise only marginally so if you have not given a name to your queue, so please do that.
Eg:
```
action( type="omrelp"
        name="logtoserver" # Name of queue
        ...
```

Failing this you can take out a bigger gun and buy a bigger screen for the subsequent required
screen real estate.

The bigger gun is running rsyslog in a non-daemonised fashion on the command line:
```
rsyslogd -dn
```
This will put rsyslog in debug mode and prevent it from going into the background. You will
need to stop the normal daemon from running though. I have found that systemctl is fairly
robust in keeping things running so in order to get this working I did the following:
```
systemctl disable rsyslog
systemctl stop rsyslog
```
Once you have disabled and stopped the service you will then be able to run it in debug mode.
The debug mode is rather verbose but that is better than not having any information at all.
It is particularly usefull to see if the settings that you have set, will actually get set
and it helps to give insight into what rsyslog is doing.
