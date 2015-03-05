+++
date = "2015-02-28T18:48:24-08:00"
title = "Analyzer"
+++

<a href="#" class="image fit"><img src="/images/pic07.jpg" alt="" /></a>

This section will go through additional details of how the `sequence` analyzer reduces 100 of 1000's of raw log messages down to just 10's of unique patterns, and then determining how to label the individual tokens. The goal is to reduce the time to write parsing rules by 50-75%.

Here are some preliminary results. Below, we analyzed 2 files. The first is a file with over 200,000 sshd messages. The second is a file with a mixture of ASA, sshd, sudo and su log messages. It contains almost 450,000 messages. By running the analyzer over these logs, the pure sshd log file returned 45 individual patterns, and the second returned 103 unique patterns.

```
$ go run sequence.go analyze -i ../../data/sshd.all -o sshd.analyze
Analyzed 212897 messages, found 45 unique patterns, 45 are new.

$ go run sequence.go analyze -i ../../data/asasshsudo.log -o asasshsudo.analyze
Analyzed 447745 messages, found 103 unique patterns, 103 are new.
```

And the output file has entries such as:

```
%msgtime% %apphost% %appname% [ %sessionid% ] : %status% %method% for %srcuser% from %srcipv4% port %srcport% ssh2
# Jan 15 19:39:26 irc sshd[7778]: Accepted password for jlz from 108.61.8.124 port 57630 ssh2

%msgtime% %appipv4% %appname% : %action% outbound %protocol% connection %sessionid% for %string% : %srcipv4% / %srcport% ( %ipv4% / %integer% ) to %string% : %dstipv4% / %dstport% ( %ipv4% / %integer% )
# 2012-04-05 18:46:18   172.23.0.1  %ASA-6-302013: Built outbound TCP connection 1424575 for outside:10.32.0.100/80 (10.32.0.100/80) to inside:172.23.73.72/2522 (10.32.0.1/54702)

%msgtime% %apphost% %appname% : %string% : tty = %string% ; pwd = %string% ; user = %srcuser% ; command = %command% - %string%
# Jan 15 14:09:11 irc sudo:    jlz : TTY=pts/1 ; PWD=/home/jlz ; USER=root ; COMMAND=/bin/su - irc
```
As you can see, the output is not 100%, but it gets us pretty close. Once the analyst goes through and updates the rules, he/she can re-run the analyzer anytime with any file to determine if there's new patterns. For example, below, we ran the sshd log file with an existing pattern file, and got 4 new log patterns.

```
$ go run sequence.go analyze -i ../../data/sshd.all -p ../../patterns/sshd.txt -o sshd.analyze
Analyzed 212897 messages, found 39 unique patterns, 4 are new.
```

The _analyzer_ is performed in two separate steps:

* Identifying the unique patterns
* Determining the correct fields

### Identifying the Unique Patterns

Analyzer builds an analysis tree that represents all the _sequences_ from the _scanners_. It can be used to determine all of the unique patterns for a large body of messages.

It's based on a single basic concept, that for multiple log messages, if tokens in the same position shares one same parent and one same child, then the tokens in that position is likely variable string, which means it's something we can extract. For example, take a look at the following two messages:

```
Jan 12 06:49:42 irc sshd[7034]: Accepted password for root from 218.161.81.238 port 4228 ssh2
Jan 12 14:44:48 jlz sshd[11084]: Accepted publickey for jlz from 76.21.0.16 port 36609 ssh2
```

The first token of each message is a timestamp, and the 3rd token of each message is the literal "sshd". For the literals "irc" and "jlz", they both share a common parent, which is a timestamp. They also both share a common child, which is "sshd". This means token in between these, the 2nd token in each message, likely represents a variable token in this message type. In this case, "irc" and "jlz" happens to
represent the syslog host.

Looking further down the message, the literals "password" and "publickey" also share a common parent, "Accepted", and a common child, "for". So that means the token in this position is also a variable token (of type TokenString).

You can find several tokens that share common parent and child in these two messages, which means each of these tokens can be extracted. And finally, we can determine that the single pattern that will match both is:

```
%time% %string% sshd [ %integer% ] : Accepted %string% for %string% from %ipv4% port %integer% ssh2
```

If later we add another message to this mix:

```
Jan 12 06:49:42 irc sshd[7034]: Failed password for root from 218.161.81.238 port 4228 ssh2
```

The Analyzer will determine that the literals "Accepted" in the 1st message, and "Failed" in the 3rd message share a common parent ":" and a common child "password", so it will determine that the token in this position is also a variable token. After all three messages are analyzed, the final pattern that will match all three
messages is:

```
%time% %string% sshd [ %integer% ] : %string% %string% for %string% from %ipv4% port %integer% ssh2
```

By applying this concept, we can effectively identify all the unique patterns in a log file.

### Determining the Correct Fields

Now that we have the unique patterns, we will scan the tokens to determine which labels we should apply to them. 

System and network logs are mostly free form text. There's no specific patterns to any of them. So it's really difficult to determine how to label specific parts of the log message automatically. However, over the years, after looking at so many system and network log messages, some patterns will start to emerge. 

There's no "machine learning" here. This section is all about codifying these human learnings. I've created the following 6 rules to help label tokens in the log messages. By no means are these rules perfect. They are at best just guesses on how to label. But hopefully they can get us 75% of the way there and we human can just take it the rest of the way.

#### 0. Parsing Email and Hostname Formats

This is technically not a labeling step. Before we actually start the labeling process, we wanted to first parse out a couple more formats like email and host names. The message tokenizer doesn't recognize these because they are difficult to parse and will slow down the tokenizer. These specific formats are also not needed by the parser. So because the analyzer doesn't care about performance as much, we can do this as post-processing step.

To recognize the hostname, we try to match the "effective TLD" using the [xparse/etld](https://github.com/surge/xparse/tree/master/etld) package. It is an effective TLD matcher that returns the length of the effective domain name for the given string. It uses the data set from https://www.publicsuffix.org/list/effective_tld_names.dat.

#### 1. Recognizing Syslog Headers

First we will try to see if we can regonize the syslog headers. We try to recogize both RFC5424 and RFC3164 syslog headers:

```
	// RFC5424
	// - "1 2003-10-11T22:14:15.003Z mymachine.example.com evntslog - ID47 ..."
	// - "1 2003-08-24T05:14:15.000003-07:00 192.0.2.1 myproc 8710 - ..."
	// - "1 2003-10-11T22:14:15.003Z mymachine.example.com su - ID47 ..."
	// RFC3164
	// - "Oct 11 22:14:15 mymachine su: ..."
	// - "Aug 24 05:34:00 CST 1987 mymachine myproc[10]: ..."
	// - "jan 12 06:49:56 irc last message repeated 6 times"
```

If the sequence pattern matches any of the above sequence, then we assume the first few tokens belong to the syslog header.

#### 2. Marking Key and Value Pairs

The next step we perform is to mark known "keys". There are two types of keys. First, we identify any token before the "=" as a key. For example, the message `fw=TOPSEC priv=6 recorder=kernel type=conn` contains 4 keys: `fw`, `priv`, `recorder` and `type`. These keys should be considered string literals, and should not be extracted. However, they can be used to determine how the value part should be labeled.

The second types of keys are determined by keywords that often appear in front of other tokens, I call these **prekeys**. For example, we know that the prekey `from` usually appears in front of any source host or IP address, and the prekey `to` usually appears in front of any destination host or IP address. Below are some examples of these prekeys.

```
from 		= [ "%srchost%", "%srcipv4%" ]
port 		= [ "%srcport%", "%dstport%" ]
proto		= [ "%protocol%" ]
sport		= [ "%srcport%" ]
src 		= [ "%srchost%", "%srcipv4%" ]
to 			= [ "%dsthost%", "%dstipv4%", "%dstuser%" ]
```

#### 3. Labeling "Values" by Their Keys

Once the keys are labeled, we can label the values based on the mapping described above. For key/value pairs, we try to recognize both `key=value` or `key="value"` formats (or other quote characters like ' or <).

For the prekeys, we try to find the value token within 2 tokens of the key token. That means sequences such as `from 192.168.1.1` and `from ip 192.168.1.1` will identify `192.168.1.1` as the `%srcipv4%` based on the above mapping, but we will miss `from ip address 192.168.1.1`. 

#### 4. Identifying Known Keywords

Within most log messages, there are certain keywords that would indicate what actions were performed, what the state/status of the action was, and what objects the actions were performed on. CEE had a list that it identified, so I copied the list and added some of my own. 

```
action = [
	"access",
	"alert",
	"allocate",
	"allow",
	.
	.
	.
]

status = [
	"accept",
	"error",
	"fail",
	"failure",
	"success"
]

object = [
	"account",
	"app",
	"bios",
	"driver",
	.
	.
	.
]
```

In our labeling process, we basically goes through and identify all the string literals that are NOT marked as keys, and perform a [porter2 stemming operation](https://github.com/surge/porter2) on the literal, then compare to the above list (which is also porter2 stemmed).

If a literal matches one of the above lists, then the corresponding label (`action`, `status`, `object`, `srcuser`, `method`, or `protocol`) is applied.

#### 5. Determining Positions of Specific Types

In this next step, we are basically looking at the position of where some of the token types appear. Specifically, we are looking for `%time%`, `%url%`, `%mac%`, `%ipv4%`, `%host%`, and `%email%` tokens. Assuming the labels have not already been taken with the previous rules, the rules are as follows:

* The first %time% token is labeled as %msgtime%
* The first %url% token is labeled as %object%
* The first %mac% token is labeled as %srcmac% and the second is labeld as %dstmac%
* The first %ipv4% token is labeled as %srcipv4% and the second is labeld as %dstipv4%
* The first %host% token is labeled as %srchost% and the second is labeld as %dsthost%
* The first %email% token is labeled as %srcemail% and the second is labeld as %dstemail%

#### 6. Scanning for ip/port or ip:port Pairs

Finally, after all that, we scan through the sequence again, and identify any numbers that follow an IP address, but separated by either a "/" or ":". Then we label these numbers as either `%srcport%` or `%dstport%` based on how the previous IP address is labeled.