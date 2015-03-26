+++
date = "2015-02-28T18:48:24-08:00"
title = "Introduction"
weight = 10
series_weight = 10
series = [ "overview" ]
+++

<a href="#" class="image fit"><img src="/images/pic05.jpg" alt="Obligatory Log Picture" /></a>

`sequence` is a _high performance sequential log analyzer and parser_. It _sequentially_ goes through a log message, _parses_ out the meaningful parts, without the use regular expressions. It can achieve _high performance_ parsing of **100,000 - 200,000 messages per second (MPS)** without the need to separate parsing rules by log source type.

**`sequence` is currently under active development and should be considered unstable until further notice.**

**If you have a set of logs you would like me to test out, please feel free to [open an issue](https://github.com/strace/sequence/issues) and we can arrange a way for me to download and test your logs.**

### Motivation

Log messages are notoriusly difficult to parse because they all have different formats. Industries (see Splunk, ArcSight, Tibco LogLogic, Sumo Logic, Logentries, Loggly, LogRhythm, etc etc etc) have been built to solve the problems of parsing, analyzing and understanding log messages.

Let's say you have a bunch of log files you like to parse. The first problem you will typically run into is you have no way of telling how many DIFFERENT types of messages there are, so you have no idea how much work there will be to develop rules to parse all the messages. Not only that, you have hundreds of thousands, if not  millions of messages, in front of you, and you have no idea what messages are worth parsing, and what's not.

The typical workflow is develop a set of regular expressions and keeps testing against the logs until some magical moment where all the logs you want parsed are parsed. Ask anyone who does this for a living and they will tell you this process is long, frustrating and error-prone.

Sequence is developed to make analyzing and parsing log messages a lot easier and faster.

### Existing Approaches

The industry has came up with a couple of different approaches to solving the log parsing problem. In a way, you can say the log parsing problem has been solved, because analysts have a lot of different tools to choose from when they need to understand logs.

Commercially, there are companies such as Splunk, ArcSight, Tibco LogLogic, SumoLogic, LogEntries, Loggly, LogRhythm, etc etc etc that can provide you either an on-premise or in-the-cloud (SaaS) solution. They provide different capabilities and feature sets depending on the primary use case you are targetting.

Open source wise, you have tools such as ElasticSearch, Greylog2, OSSIM, and a few others that wants to provide you end-to-end capabilities similar to the commercial offerings. There are also libraries such as [liblognorm](http://www.liblognorm.com/) and [logstash](http://logstash.net/) you can use to build your own tools.

And then there's Fedora's [Project Lumberjack](https://fedorahosted.org/lumberjack/), which "is an open-source project to update and enhance the event log architecture" and "aims to improve the creation and standardize the content of event logs by implementing the concepts and specifications proposed by the ​Common Event Expression (CEE)."

Unfortunately all of these tools have one or more of the following problems.

_First_, it looks like many of these open source efforts have all been abandoned or put in hibernation, and haven't been updated since 2012 or 2013. liblognrom did put out [a couple of updates](http://www.liblognorm.com/news/) in the past couple of years. 

It is understandable. Log parsing is **BORING**. I mean, who wants to sit there and stare at logs all day and try to come up with regular expressions or other types of parsing rules? LogLogic used to have a team of LogLabs analysts that did that, and I have to say I truly appreciated their effort and patience, because I cannot do that.

_Second_, many of these commercial and open source tools uses regular expression to parse log messages. This approach is widely adopted because regular expression (regex) is a known quantity. Many administrators already know regex to some extend, and tools are widely available to interpret regex. In the early days of log analysis, Perl was used most often, so most rules you see are written in PCRE, or Perl Compatible Regular Expression. However, the process of writing regex rules is long, frustrating, and error-prone. 

Even after you have developed a set of regular expressions that match the original set of messages, if new messages come in, you will have to determine which of the new messages need to be parsed. And if you develop a new set of regular expressions to parse those new messages, you still have no idea if the regular expressions will conflict with the ones you wrote before. If you write your regex parsers too liberally, it can easily parse the wrong messages.

_Third_, even after the regex rules are written, the performance is far from acceptable. This is mainly due to the fact that there's no way to match multiple regular expressions at the same time. The engine would have to go through each individual rule separately. It can typically parse several thousands messages per second. Given enough CPU resources on a large enough machine, regex parsers can probably parse tens of thousands of messages per second. Even to achieve this type of performance, you will likely need to limit the number of regular expressions the parser has. The more regex rules, the slower the parser will go.

To work around this performance issue, companies have tried to separate the regex rules for different log message types into different parsers. For example, they will have a parser for Cisco ASA logs, a parser for sshd logs, a parser for Apache logs, etc etc. And then they will require the analysts to tell them which parser to use (usually by indicating the log source type of the originating IP address or host.)

_Last but not least_, none of the existing tools can help analysts determine what patterns to write in order to parse their log files. Large companies can sometimes generate hundreds of gigabytes, if not terabytes, of data and billions of log messages per day. Sometimes it will take a team of analysts hundreds of hours to comb through log files, develop regex rules, test for conflicts and then repeat this whole process. 

