+++
date = "2015-02-28T18:48:24-08:00"
title = "Scanner"
+++

> In computer science, lexical analysis is the process of converting a sequence of characters into a sequence of tokens, i.e. meaningful character strings. A program or function that performs lexical analysis is called a lexical analyzer, lexer, tokenizer, or scanner. - [Wikipedia](http://en.wikipedia.org/wiki/Lexical_analysis)

In `sequence`, the first critical function to understanding a log message is to convert it into a sequence of valid tokens, i.e., **meaningful** character strings. This function is called a _scanner_, but it can be called a _lexical analyzer_, _lexer_, or _tokenizer_. The `sequence` _scanner_ goes through log message sequentially while tokentizing each part of the message, without the use of regular expressions.

The biggest challenge to tokenizing is knowing where the break points are. Most log messages are free-form text, which means there's no common structure to them. A log message _scanner_ or _tokenizer_ (we will use these terms interchangeably) must understand common components such as timestamp, URL, hex strings, IP addresses (v4 or v6), and mac addresses, so it can break the messages into **meaningful** tokens.

There are two reasons for the emphasis on **meaningful**. First, due to the unstructured nature of most log messages, a scanner cannot depend on whitespaces to separate meaningful parts of the message. As an example, most timestamps in log messages have whitespaces, such as `01/02 03:04:05PM '06 -0700`. This semantically should be a single token. But if a scanner tokenizes based on whitespace, then it will be broken into 4 differnt parts, including `01/02`, `03:04:05PM`, `'06` and `-0700`. This would obviously be wrong.

A scanner also cannot depend on punctuations to tokenize the log message. In the above timestamp example, if you count `/` and `:` as punctuations, then you would end up with 4 different tokens, including `01`, `02 03`, `04`, and `05PM '06 -0700`. 

Same if you were to look at an IPv4 address such as "127.0.0.1", or IPv6 address such as "f0f0:f::1", or MAC address such as "00:04:c1:8b:d8:82", or a hex signature such as "de:ad:be:ef:74:a6:bb:45:45:52:71:de:b2:12:34:56". All these should be their own tokens. But if a scanner depends on only whitespace or punctuations to tokenize a message, these valid tokens would all be broken apart.

Second, understanding the types of tokens will help the _parser_ distinguish between different log messages. If two messages are almost identical but one has a hostname and the other has an IP address, the _scanner_ can let the _parser_ know, and the _parser_ would the choose the correct pattern for the message.

The `sequence` scanners currently automatically recognizes [9 different token types](/manual/tokens) and [42 different time stamps](/manual/timeformats) .

### Scanners

`sequence` implemented two scanners. First is a **general scanner** which is a sequential lexical analyzer that breaks unstructured log messages into sequences of tokens. This scanner is mianly used for most system and network log messages that are free-form unstructured text. 

As an example, the following log message can be tokenized into the sequence of tokens below. As you can see, one cannot depend on white spaces to tokenize, as the timestamp would be broken into 3 parts; nor can one use punctuations like ";" or ":", as they would break the log mesage into useless parts. 

```
jan 14 10:15:56 testserver sudo:    gonner : tty=pts/3 ; pwd=/home/gonner ; user=root ; command=/bin/su - ustream

  #   0: { Field="funknown", Type="ts", Value="jan 14 10:15:56" }
  #   1: { Field="funknown", Type="literal", Value="testserver" }
  #   2: { Field="funknown", Type="literal", Value="sudo" }
  #   3: { Field="funknown", Type="literal", Value=":" }
  #   4: { Field="funknown", Type="literal", Value="gonner" }
  #   5: { Field="funknown", Type="literal", Value=":" }
  #   6: { Field="funknown", Type="literal", Value="tty" }
  #   7: { Field="funknown", Type="literal", Value="=" }
  #   8: { Field="funknown", Type="string", Value="pts/3" }
  #   9: { Field="funknown", Type="literal", Value=";" }
  #  10: { Field="funknown", Type="literal", Value="pwd" }
  #  11: { Field="funknown", Type="literal", Value="=" }
  #  12: { Field="funknown", Type="string", Value="/home/gonner" }
  #  13: { Field="funknown", Type="literal", Value=";" }
  #  14: { Field="funknown", Type="literal", Value="user" }
  #  15: { Field="funknown", Type="literal", Value="=" }
  #  16: { Field="funknown", Type="string", Value="root" }
  #  17: { Field="funknown", Type="literal", Value=";" }
  #  18: { Field="funknown", Type="literal", Value="command" }
  #  19: { Field="funknown", Type="literal", Value="=" }
  #  20: { Field="funknown", Type="string", Value="/bin/su" }
  #  21: { Field="funknown", Type="literal", Value="-" }
  #  22: { Field="funknown", Type="literal", Value="ustream" }
```

Second is a **JSON scanner**, implemented by `ScanJson()`, that scans JSON messages and convert them into a _sequence_ that can be parsed by the _parser_. 

`ScanJson()` will flatten a json string into key=value pairs, and it performs the following transformation:

- all {, }, [, ], ", characters are removed
- colon between key and value are changed to "="
- nested objects have their keys concatenated with ".", so a json string like `"userIdentity": {"type": "IAMUser"}` will be returned as `userIdentity.type=IAMUser`
- arrays are flattened by appending an index number to the end of the key, starting with 0, so a json string like `{"value":[{"open":"2014-08-16T13:00:00.000+0000"}]}` will be returned as `value.0.open=2014-08-16T13:00:00.000+0000`
- skips any key that has an empty value, so json strings like `"reference":""` or `"filterSet": {}` will not show up in the Sequence

For example:

```
{"EventTime":"2014-08-16T12:45:03-0400","URI":"myuri","uri_payload":{"value":[{"open":"2014-08-16T13:00:00.000+0000","close":"2014-08-16T23:00:00.000+0000","isOpen":true,"date":"2014-08-16"}],"Count":1}}

  #   0: { Field="funknown", Type="literal", Value="EventTime" }
  #   1: { Field="funknown", Type="literal", Value="=" }
  #   2: { Field="funknown", Type="time", Value="2014-08-16T12:45:03-0400" }
  #   3: { Field="funknown", Type="literal", Value="URI" }
  #   4: { Field="funknown", Type="literal", Value="=" }
  #   5: { Field="funknown", Type="literal", Value="myuri" }
  #   6: { Field="funknown", Type="literal", Value="uri_payload.value.0.open" }
  #   7: { Field="funknown", Type="literal", Value="=" }
  #   8: { Field="funknown", Type="time", Value="2014-08-16T13:00:00.000+0000" }
  #   9: { Field="funknown", Type="literal", Value="uri_payload.value.0.close" }
  #  10: { Field="funknown", Type="literal", Value="=" }
  #  11: { Field="funknown", Type="time", Value="2014-08-16T23:00:00.000+0000" }
  #  12: { Field="funknown", Type="literal", Value="uri_payload.value.0.isOpen" }
  #  13: { Field="funknown", Type="literal", Value="=" }
  #  14: { Field="funknown", Type="literal", Value="true" }
  #  15: { Field="funknown", Type="literal", Value="uri_payload.value.0.date" }
  #  16: { Field="funknown", Type="literal", Value="=" }
  #  17: { Field="funknown", Type="time", Value="2014-08-16" }
  #  18: { Field="funknown", Type="literal", Value="uri_payload.Count" }
  #  19: { Field="funknown", Type="literal", Value="=" }
  #  20: { Field="funknown", Type="integer", Value="1" }
```

### Design

Tokenizers or scanners are usually implemented using finite-state machines. Each FSM (or FSA, finite state automata) understands a specific sequences of characters that make up a type of token. 

In the `sequence` scanner, there are three FSMs: Time, HexString and General.

* The Time FSM understands [42 different time stamps](/manual/timeformats). This list of time formats are commonly seen in log messages. It is also fairly easy to add to this list if needed.
* The HexString FSM is designed to understand IPv6 addresses (dead:beef:1234:5678:223:32ff:feb1:2e50 or f0f0:f::1), MAC addresses (00:04:c1:8b:d8:82), fingerprints or signatures (de:ad:be:ef:74:a6:bb:45:45:52:71:de:b2:12:34:56).
* The General FSM that recognizes URLs, IPv4 addresses, and any literal or strings.

Each character in the log message are run through all three FSMs, and the following logics are applied: 

1. If a time format is matched, that's what it will be returned. 
1. Next if a hex string is matched, it is also returned. 
  * We mark anything with 5 colon characters and no successive colons like "::" to be a MAC address. 
  * Anything that has 7 colons and no successive colons are marked as IPv6 address. 
  * Anything that has less than 7 colons but has only 1 set of successive colons like "::" are marked as IPv6 address.
  * Everything else is just a literal.
1. Finally if neither of the above matched, we return what the general FSM has matched.
  * The general FSM recognizes these quote characters: ", ' and <. If these characters are encountered, then it will consider anything between the quotes to be a single token.
  * Anything that starts with http:// or https:// are considered URLs.
  * Anything that matches 4 integer octets are considered IP addresses.
  * Anything that matches two integers with a dot in between are considered floats.
  * Anything that matches just numbers are considered integers.
  * Everything else are literals.

To achieve the performance we want, `sequence` took great pain to go through the log message once and only once. This is probably a pretty obvious technique. The more times you loop through loop through a string, the lower the performance. 

`sequence` also took great pain to ensure that there's no need to look forward or look backward in the log message to determine the current token type. If you used regular expressions to parse logs, you will likely go through parts of the log message multiple times due to back tracking or look forward, etc.

In reality though, while the scanners only looping through the log string once, and only once, it does run each character through three different FSMs. However, it is still much less expensive than looping through three times, each time checking a single FSM. 

You can [read more details](http://zhen.org/blog/sequence-optimizing-go-for-high-performance-log-scanner/) on how the scanners were able to achieve the performance.

### Performance

The `sequence` _scanner_ is able to tokenize almost 200,000 messages per second for messages averaging 136 bytes. The following performance benchmarks are run on a single 4-core (2.8Ghz i7) MacBook Pro, although the tests were only using 1 or 2 cores. The first file is a bunch of sshd logs, averaging 98 bytes per message. The second is a Cisco ASA log file, averaging 180 bytes per message. Last is a mix of ASA, sshd and sudo logs, averaging 136 bytes per message.

```
  $ ./sequence bench scan -i ../../data/sshd.all
  Scanned 212897 messages in 0.78 secs, ~ 272869.35 msgs/sec

  $ ./sequence bench scan -i ../../data/allasa.log
  Scanned 234815 messages in 1.43 secs, ~ 163827.61 msgs/sec

  $ ./sequence bench scan -i ../../data/allasassh.log
  Scanned 447745 messages in 2.27 secs, ~ 197258.42 msgs/sec
 ```

Performance can be improved by adding more cores:


```
  $ GOMAXPROCS=2 ./sequence bench scan -i ../../data/sshd.all -w 2
  Scanned 212897 messages in 0.43 secs, ~ 496961.52 msgs/sec

  $ GOMAXPROCS=2 ./sequenceo bench scan -i ../../data/allasa.log -w 2
  Scanned 234815 messages in 0.80 secs, ~ 292015.98 msgs/sec

  $ GOMAXPROCS=2 ./sequence bench scan -i ../../data/allasassh.log -w 2
  Scanned 447745 messages in 1.20 secs, ~ 373170.45 msgs/sec
```