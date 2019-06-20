# 🖥️ Log basics and Graylog

**Tags:**  `LOGS` ,  `SYSLOG` ,  `RSYSLOG` ,  `SYSLOG-NG` ,  `JOURNALD` ,  `GRAYLOG` ,  `RFC 5424` ,  `SYSTEMD` 


## Document structure:
- [Syslog](#Syslog)
- [rsyslog VS syslog-ng](#rsyslog-VS-syslog-ng)
- [syslog has several key problems](#Syslog-has-several-key-problem)
- [systemd introduces journald](#systemd-introduces-journald)
- [Graylog](#Graylog)






# Syslog

Syslog is the standard solution for logging on UNIX.  The *syslog* facility provides a single, centralized logging facility that can be used to log messages by all applications on the system. 


![]()


The syslog facility has two principal components:

- the *syslogd* daemon and
- the *syslog(3)* library function.
## syslogd

The System Log daemon, syslogd, accepts log messages from two different sources:

- a UNIX domain socket, `/dev/log`, which holds locally produced messages
- (if enabled) an Internet domain socket (UDP port 514), which holds messages sent across a TCP/IP network. (On some other UNIX implementations, the syslog socket is located at /var/run/log.)


## syslog(3)

The syslog(3) library function can be used by any process to log a message. This function, which we describe in detail in a moment, uses its supplied arguments to construct a message in a standard format that is then placed on the `/dev/log` socket for reading by syslogd.



## Message

Each message processed by syslogd has a number of attributes, including

- a **facility**, which specifies the type of program generating the message, and
- a **level**, which specifies the severity (priority) of the message.

For example, `level` & `facility` values from 2 machines (medianetlab *k8s-master* / *k8s-node*) for the “last 2 days” presented in a Graylog dashboard.

![]()


The syslogd daemon examines the facility and level of each message, and then passes it along to any of several possible destinations according to the dictates of an associated configuration file,
`/etc/syslog.conf`. Possible destinations include:

- a terminal or virtual console,
- a disk file,
- a FIFO,
- one or more (or all) logged-in users, or a process (typically
- another syslogd daemon) on another system connected via a TCP/IP network.

Sending the message to a process on another system is useful for reducing administrative overhead by consolidating messages from multiple systems to a single location. A single message may be sent to multiple destinations (or none at all), and messages with different combinations of facility and level can be targeted to different destinations or to different instances of destinations (i.e., different consoles, different disk files, and so on).








# rsyslog VS syslog-ng

*source:* [*What is the difference between syslog, rsyslog and syslog-ng?*](https://serverfault.com/questions/692309/what-is-the-difference-between-syslog-rsyslog-and-syslog-ng)

Basically, they are the same, in the way they all permit the logging of data from different types of systems in a central repository.

*The Syslog project was the very first project. It started in 1980. It is the root project to* `*Syslog*` *protocol. At this time Syslog is a very simple protocol. At the beginning it only supports UDP for transport, so that it does not guarantee the delivery of the messages.*

Next came **syslog-ng** in 1998. It extends basic `syslog` protocol with new features like:

- content-based filtering
- Logging directly into a database
- TCP for transport
- TLS encryption

Next came **rsyslog** in 2004. It extends `syslog` protocol with new features like:

- RELP Protocol support
- Buffered operation support

Let's say that today they are three concurrent projects that have grown separately upon versions, but also grown in parallel regarding what the neighbors was doing.








# Syslog has several key problems…

*source:* [*Why Journald?*](https://www.loggly.com/blog/why-journald/)


- syslog implementations write log messages to plain text files
- UNIX has great tools to work with plain text files **BUT** a lack of structure is the source of pretty much all of syslog’s problems
- finding information in large plain text files with lots of unrelated information can be difficult
- syslog implementations generally allow administrators to split up their files according to pre-defined topics, but they then end up with many smaller files and no easy way to correlate information between files
- the syslog protocol does not provide a means of separating messages by application-defined targets. For example, web servers can emit log messages per virtual host. Because syslog cannot deal with such meta information, the web servers generally write their own access logs so that the main system log is not flooded with web server status messages. This creates additional sources of possible log messages an admin has to keep in mind, with additional places where these messages are configured
- Simple plain text files also require log rotation to prevent them from becoming too large. In log rotation, existing log files are renamed and compressed. Any programs that watch syslog messages for problems have to deal with this somehow.
- As log files write messages terminated by a newline, a log message can not contain newlines. This makes it hard for programs to emit multi-line information such as backtraces when an error occurs, and log parsing software must often do a lot of work to combine log messages spread over multiple lines.






# systemd introduces journald

Journald is a system service for collecting and storing log data, introduced with systemd. It tries to make it easier for system administrators to find interesting and relevant information among an ever-increasing amount of log messages.

- In keeping with this goal, one of the main changes in journald was to replace simple plain text log files with a special file format optimized for log messages. This file format allows system administrators to access relevant messages more efficiently
- It also brings some of the power of database-driven centralized logging implementations to individual systems
- it retains full syslog compatibility by providing the same API in C, supporting the same protocol, and also forwarding plain-text versions of messages to an existing syslog implementation
- The format, as well as the journald API, allow for structured data and quick access / retrieval of messages by all fields.
- It also does away with log rotation by using a space-optimized format directly that does not require renaming files to archive entries and automatically limiting the maximum size of the journal on the disk. This removes a lot of the difficulties programs face when dealing with log files.
## journalctl

As this structured file format does not work well with standard UNIX tools that are optimized for plain text, there is a command line tool that can query these files, `journalctl(1)`. This program utilizes the journal format to give very fast access to entries filtered by date, emitting program, program PID, UID, service, or other elements. In short, the log file format of journald makes it much easier for programs to retrieve only the information they want to know about, in a structured way, with the ability to easily follow the stream of new log messages in real time.

**BUT**

- At the same time, journald does not include a well-defined remote logging implementation. Instead, it relies on existing syslog implementations to relay messages to a central log host, thereby losing most of the benefits of the new system.








# Graylog

Graylog can be used as a central point where different servers push their logs.  The sysadmin can then check their status without:

- having to “ssh“ into different hosts
- dealing with various log files
- playing with Linux commands that manipulate text.

(Setup below: 2 machines (k8s-master & k8s-node) push their logs to our graylog instance)



## Easy way to send logs to Graylog

(for Ubuntu 18.04)
You have to create a new file inside the  `/etc/rsyslog.d/`  directory, for example  `90-graylog.conf`  , add the following line:
 `*.* @[graylog_server_ip]:[input_port];RSYSLOG_SyslogProtocol23Format` 
 and restart the service:  `systemctl restart rsyslog.service` 
 
 On the graylog side, you will have to launch a new “syslog input” and specify the above port.
 
 For example:

    root@k8s-master:/etc/rsyslog.d# ls
    20-ufw.conf  21-cloudinit.conf  50-default.conf  90-graylog.conf
    
    root@k8s-master:/etc/rsyslog.d# cat 90-graylog.conf
    *.* @10.30.0.245:5140;RSYSLOG_SyslogProtocol23Format


## Search

Two parameters are needed:

- time period (relative or specific)
- search query *(If the search query is empty, all the logs are retrieved)*

After you press the search button, a **histogram** is generated where you can see the number of received logs per minute / hour / day… On the left side you can turn on/off the **fields** that are being displayed (source, facility, level…). On the right there is a table with the **messages** (logs).


click to zoom **🔎**

![]()




## Visualization types
- chart (histogram or line)
- quick values (pie chart)
- statistics (table)
- world map


click to zoom **🔎**

![]()





## How to add a line graph

This is “hackish” and not recommended. Graylog is not for this job.

Edit one of your machine’s crontab and add the following line:


    * * * * * bash -c "uptime | logger -t uptime"

This will silently run every minute and log the output of the  `uptime`  command, something like this:


    14:17:10 up 24 days,  3:52,  1 user,  load average: 1.25, 1.40, 1.72

To extract the first value of the load average, we then go to graylog > inputs > manage extractors and create a new extractor with the following regular expression:
 `^.*load average: ([\d.,]+),.*$` 

If you have set up your host to send the logs to Graylog, you can see them by applying a simple query like  `application_name:uptime`  and you can also see that in the “fields” panel there is also a field named  `load_average_test2`  which is the extracted value for every log message:

click to zoom **🔎**

![]()


To generate the line graph we will have to modify our query in a way that only the uptime values of **ONE** source are being retrieved, for example  `application_name:uptime && source: k8s-node` 
Second step: from the “fields” panel, select the load_average

click to zoom **🔎**

![]()




## Reasons to avoid using Graylog


- limitations with visualization options (graylog is basically good only at histograms and pie-charts).
- minimum time resolution for graphs: 1 minute
- difficult to manage time in dashboards in a unified way. Every visualization has a “Search relative time” setting and  it’s very easy to end up with graphs that don’t display information for the same time range.
- there is no way to control the colors used and there are also inconsistencies between graphs (same value has different colors in different graphs of the same dashboard), for example:
![]()

- you can’t combine metrics from different sources to the same graph with different colors. You have to use separate queries / graphs, for example:  `application_name:uptime AND source:k8s-master`  /  `application_name:uptime AND source:k8s-node` 
![]()


