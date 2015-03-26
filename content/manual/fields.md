+++
date = "2015-02-28T18:48:24-08:00"
title = "Predefined Fields"
weight = 110
series_weight = 20
series = [ "configuration" ]
+++

<a href="#" class="image fit"><img src="/images/pic04.jpg" alt="" /></a>

The goal of any log parser is to extract meaningful parts from a log message and then semantically tag them so other tools can perform analysis on the tags and values. In `sequence`, these tags are called _fields_.

`sequence` predefines a set of fields based on the analysis of many system and network logs. However, analysts can add their own fields to the file as well. When adding a new field, user can add a field of the format `field:type` to the `parser` portion of the configuration file in the `fields` array. 

```
[parser]
fields = [
	"msgid:string",				# The message identifier
	.
	.
	.
	"userdefined1:string"			# This is user defined field #1
]
```

The field name can be any alphanumeric `a-zA-Z0-9_]` string chosen by the user. The type can be any of the pre-defined [token types](/manual/tokens). The type is considered to be the default type of the field. When a field is defined in a message pattern, if no type is defined in pattern, then the default type is used. This helps reduce the amount of typing analysts have to do.

It's highly recommended that all user defined fields are added to the end of the array, after the predefined fields. A comment explaning what it is and how it might be used is also highly recommended.

The following list of fields are predefined in the [sequence.toml](/manual/config.md) file.

| Field | Type | Description |
|-------|------|-------------|
| msgid | string |  The message identifier |
| msgtime | time |  The timestamp that’s part of the log message |
| severity | integer |  The severity of the event, e.g., Emergency, … |
| priority | integer |  The pirority of the event |
| apphost | string |  The hostname of the host where the log message is generated |
| appip | ipv4 |  The IP address of the host where the application that generated the log message is running on. |
| appvendor | string |  The type of application that generated the log message, e.g., Cisco, ISS |
| appname | string |  The name of the application that generated the log message, e.g., asa, snort, sshd |
| srcdomain | string |  The domain name of the initiator of the event, usually a Windows domain |
| srczone | string |  The originating zone |
| srchost | string |  The hostname of the originator of the event or connection. |
| srcip | ipv4 |  The IPv4 address of the originator of the event or connection. |
| srcipnat | ipv4 |  The natted (network address translation) IP of the originator of the event or connection. |
| srcport | integer |  The port number of the originating connection. |
| srcportnat | integer |  The natted port number of the originating connection. |
| srcmac | mac |  The mac address of the host that originated the connection. |
| srcuser | string |  The user that originated the session. |
| srcuid | integer |  The user id that originated the session. |
| srcgroup | string |  The group that originated the session. |
| srcgid | integer |  The group id that originated the session. |
| srcemail | string |  The originating email address |
| dstdomain | string |  The domain name of the destination of the event, usually a Windows domain |
| dstzone | string |  The destination zone |
| dsthost | string |  The hostname of the destination of the event or connection. |
| dstip | ipv4 |  The IPv4 address of the destination of the event or connection. |
| dstipnat | ipv4 |  The natted (network address translation) IP of the destination of the event or connection. |
| dstport | integer |  The destination port number of the connection. |
| dstportnat | integer |  The natted destination port number of the connection. |
| dstmac | mac |  The mac address of the destination host. |
| dstuser | string |  The user at the destination. |
| dstuid | integer |  The user id that originated the session. |
| dstgroup | string |  The group that originated the session. |
| dstgid | integer |  The group id that originated the session. |
| dstemail | string |  The destination email address |
| protocol | string |  The protocol, such as TCP, UDP, ICMP, of the connection |
| iniface | string |  The incoming interface |
| outiface | string |  The outgoing interface |
| policyid | integer |  The policy ID |
| sessionid | integer |  The session or process ID |
| object | string |  The object affected. |
| action | string |  The action taken |
| command | string |  The command executed |
| method | string |  The method in which the action was taken, for example, public key or password for ssh |
| status | string |  The status of the action taken |
| reason | string |  The reason for the action taken or the status returned |
| bytesrecv | integer |  The number of bytes received |
| bytessent | integer |  The number of bytes sent |
| pktsrecv | integer |  The number of packets received |
| pktssent | integer |  The number of packets sent |
| duration | integer | The duration of the session |
