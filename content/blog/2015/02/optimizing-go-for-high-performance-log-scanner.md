+++
Categories = ["scanner", "golang"]
date = "2015-02-13T01:03:08-08:00"
title = "Optimizing Go For the High Performance Log Scanner"
+++

### tl;dr

* The `sequence` scanner is designed to tokenize free-form log messages.
  * It can scan between 200K to 500K log messages per second depending on message size and core count.
  * It recognizes time stamps, hex strings, IP (v4, v6) addresses, URLs, MAC addresses, integers and floating point numbers.
  * The design is based mostly on finite-state machines.
* The performance was achieved by the following techniques:
  1. Go Through the String Once and Only Once
  2. Avoid Indexing into the String
  3. Reduce Heap Allocation
  4. Reduce Data Copying
  5. Mind the Data Struture
  6. Avoid Interfaces If Possible
  7. Find Ways to Short Circuit Checks

### Background

> In computer science, lexical analysis is the process of converting a sequence of characters into a sequence of tokens, i.e. meaningful character strings. A program or function that performs lexical analysis is called a lexical analyzer, lexer, tokenizer, or scanner. - [Wikipedia](http://en.wikipedia.org/wiki/Lexical_analysis)

One of the most critical functions in the `sequence` parser is the message tokenization. At a very high level, message tokenization means taking a single log message and breaking it into a list of tokens. 

#### Functional Requirements

The challenge is knowing where the token break points are. Most log messages are free-form text, which means there's no common structure to them. 

As an example, the following log message can be tokenized into the sequence of tokens below. As you can see, one cannot depend on white spaces to tokenize, as the timestamp would be broken into 3 parts; nor can one use punctuations like ";" or ":", as they would break the log mesage into useless parts. 

```
jan 14 10:15:56 testserver sudo:    gonner : tty=pts/3 ; pwd=/home/gonner ; user=root ; command=/bin/su - ustream

  #   0: { Field="%funknown%", Type="%ts%", Value="jan 14 10:15:56" }
  #   1: { Field="%funknown%", Type="%literal%", Value="testserver" }
  #   2: { Field="%funknown%", Type="%literal%", Value="sudo" }
  #   3: { Field="%funknown%", Type="%literal%", Value=":" }
  #   4: { Field="%funknown%", Type="%literal%", Value="gonner" }
  #   5: { Field="%funknown%", Type="%literal%", Value=":" }
  #   6: { Field="%funknown%", Type="%literal%", Value="tty" }
  #   7: { Field="%funknown%", Type="%literal%", Value="=" }
  #   8: { Field="%funknown%", Type="%string%", Value="pts/3" }
  #   9: { Field="%funknown%", Type="%literal%", Value=";" }
  #  10: { Field="%funknown%", Type="%literal%", Value="pwd" }
  #  11: { Field="%funknown%", Type="%literal%", Value="=" }
  #  12: { Field="%funknown%", Type="%string%", Value="/home/gonner" }
  #  13: { Field="%funknown%", Type="%literal%", Value=";" }
  #  14: { Field="%funknown%", Type="%literal%", Value="user" }
  #  15: { Field="%funknown%", Type="%literal%", Value="=" }
  #  16: { Field="%funknown%", Type="%string%", Value="root" }
  #  17: { Field="%funknown%", Type="%literal%", Value=";" }
  #  18: { Field="%funknown%", Type="%literal%", Value="command" }
  #  19: { Field="%funknown%", Type="%literal%", Value="=" }
  #  20: { Field="%funknown%", Type="%string%", Value="/bin/su" }
  #  21: { Field="%funknown%", Type="%literal%", Value="-" }
  #  22: { Field="%funknown%", Type="%literal%", Value="ustream" }
```

So a log message _scanner_ or _tokenizer_ (we will use these terms interchangeably) must understand common components such as timestamp, URL, hex strings, IP addresses (v4 or v6), and mac addresses, so it can break the messages into *meaningful components*.

#### Performance Requirements

From a performance requirements perspective, I really didn't start out with any expectations. However, after achieving 100-200K MPS for parsing (not just tokenizing), I have a strong desire to keep the performance at that level. So the more I can optimize the scanner to tokenize faster, the more head room I have for parsing.

One may ask, who can POSSIBLY use such performance? Many organizations that I know are generating between 50-100M messages per second (MPS), that's only 1,200 MPS. Some larger organizations I know are generating 60GB of Bluecoat logs per day, **8 years ago**!! That's a good 3,000 MPS assuming an average of 250 bytes per message. Even if log rate grows at 15%, that's still only 10K MPS today.

To run through an example, [at EMC, 1.4 billion log messages are generated daily on average, at a rate of one terabyte a day](http://www.covert.io/research-papers/security/Beehive%20-%20Large-Scale%20Log%20Analysis%20for%20Detecting%20Suspicious%20Activity%20in%20Enterprise%20Networks.pdf). That's 16,200 messages per second, and about 714 bytes per message. (Btw, what system can possibly generate messages that are 714 bytes long? That's crazy and completely inefficient!) These EMC numbers are from 2013, so they have likely increased by now.

The `sequence` parser, with a single CPU core, can process about 270,000 MPS for messages averaging 98 bytes. Assuming the performance is linear compare to the message size (which is pretty close to the truth), we can process 37,000 MPS for messages averaging 714 bytes. That's just enough to parse the 16,2000 MPS, with a little head room to do other types of analysis or future growth. 

Obviously one can throw more hardware at solving the scale problem, but then again, why do that if you don't need to. Just because you have the hardware doesn't mean you should waste the money! Besides, there are much more interesting analytics problems your hardware can be used for than just tokenizing a message.

In any case, I want to squeeze every oz of performance out of the scanner so I can have more time in the back to parse and analyze. So let's set a goal of keeping at least 200,000 MPS for 100 bytes per message (BPM).

Yes, go ahead and tell me I shouldn't worry about micro-optimization, because this post is all about that. :)

### Sequence Scanner

In the `sequence` package, we implemented a general log message scanner, called [GeneralScanner](https://github.com/strace/sequence/blob/master/scanner.go). GeneralScanner is a sequential lexical analyzer that breaks a log message into a sequence of tokens. It is sequential because it goes through log message sequentially tokentizing each part of the message, without the use of regular expressions. The scanner currently recognizes time stamps, hex strings, IP (v4, v6) addresses, URLs, MAC addresses, integers and floating point numbers.

This implementation was able to achieve both the functional and performance requirements. The following performance benchmarks are run on a single 4-core (2.8Ghz i7) MacBook Pro, although the tests were only using 1 or 2 cores. The first file is a bunch of sshd logs, averaging 98 bytes per message. The second is a Cisco ASA log file, averaging 180 bytes per message. Last is a mix of ASA, sshd and sudo logs, averaging 136 bytes per message.

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

#### Concepts 

To understand the scanner, you have to understand the following concepts that are part of the package.

- A _Token_ is a piece of information extracted from the original log message. It is a struct that contains fields for _TokenType_, _FieldType_, and _Value_.

- A _TokenType_ indicates whether the token is a literal string (one that does not change), a variable string (one that could have different values), an IPv4 or IPv6 address, a MAC address, an integer, a floating point number, or a timestamp.

- A _FieldType_ indicates the semantic meaning of the token. For example, a token could be a source IP address (%srcipv4%), or a user (%srcuser% or %dstuser%), an action (%action%) or a status (%status%).

- A _Sequence_ is a list of Tokens. It is the key data structure consumed and returned by the _Scanner_, _Analyzer_, and the _Parser_.

Basically, the scanner takes a log message string, tokenizes it and returns a _Sequence_ with the recognized _TokenType_ marked. This _Sequence_ is then fed into the analyzer or parser, and the analyzer or parser in turn returns another _Sequence_ that has the recognized _FieldType_ marked.

#### Design

Tokenizers or scanners are usually implemented using finite-state machines. Each FSM (or FSA, finite state automata) understands a specific sequences of characters that make up a type of token. 

In the `sequence` scanner, there are three FSMs: Time, HexString and General.

* The Time FSM understands a list of [time formats](https://github.com/strace/sequence/blob/master/time.go). This list of time formats are commonly seen in log messages. It is also fairly easy to add to this list if needed.
* The HexString FSM is designed to understand IPv6 addresses (dead:beef:1234:5678:223:32ff:feb1:2e50 or f0f0:f::1), MAC addresses (00:04:c1:8b:d8:82), fingerprints or signatures (de:ad:be:ef:74:a6:bb:45:45:52:71:de:b2:12:34:56).
* The General FSM that recognizes URLs, IPv4 addresses, and any literal or strings.

Each character in the log string are run through all three FSMs. 

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

#### Performance

To achieve the performance requirements, the following rules and optimizations are followed. Some of these are Go specific, and some are general recommendations.

**1. Go Through the String Once and Only Once**

This is a hard requirement, otherwise we can't call this project a _sequential_ parser. :)

This is probably a pretty obvious technique. The more times you loop through loop through a string, the lower the performance. If you used regular expressions to parse logs, you will likely go through parts of the log message multiple times due to back tracking or look forward, etc.

I took great pain to ensure that I don't need to look forward or look backward in the log string to determine the current token type, and I think the effort paid off.

In reality though, while I am only looping through the log string once, and only once, I do run each character through three different FSMs. However, it is still much less expensive than looping through three times, each time checking a single FSM. However, the more FSMs I run the characters through, the slower it gets. 

This was apparently when I [updated the scanner to support IPv6 and hex strings](https://github.com/strace/sequence/commit/a5447814f43b4b9b7e804b14dde38e88fd53e6d0). I tried a couple of different approaches. First, I added an IPv6 specific FSM. So in addition to the original time, mac and general FSMs, there are now 4. That dropped performance by like 15%!!! That's just unacceptable.

The second approach, which is the one I checked in, combines the MAC, IPv6 and general hex strings into a single FSM. That helped somewhat. I was able to regain about 5% of the performance hit. However, because I can no longer short circuit the MAC address check (by string length and colon positions), I was still experiencing a 8-10% hit. 

What this means is that for most tokens, instead of checking just 2 FSMs because I can short circuit the MAC check pretty early, I have to now check all 3 FSMs.

So the more FSMs, the more comlicated the FSMs, the more performance hits there will be.

**2. Avoid Indexing into the String**

This is really a Go-specific recommentation. Each time you index into a slice or string, Go will perform bounds checking for you, which means there's extra operations it's doing, and also means lower performance. As an example, here are results from two benchmark runs. The first is with bounds checking enabled, which is default Go behavior. The second disables bounds checking.


```
  $ go run ./sequence.go bench scan -i ../../data/sshd.all
  Scanned 212897 messages in 0.79 secs, ~ 268673.91 msgs/sec

  $ go run -gcflags=-B ./sequence.go bench scan -i ../../data/sshd.all
  Scanned 212897 messages in 0.77 secs, ~ 277479.58 msgs/sec
```

The performance difference is approximately 3.5%! However, while it's fun to see the difference, I would never recommend disable bounds checking in production. So the next best thing is to remove as many operations that index into a string or slice as possible. Specifically:

1. Use "range" in the loops, e.g. `for i, r := range str` instead of `for i := 0; i < len(str); i++ { if str[i] == ... }`
1. If you are checking a specific character in the string/slice multiple times, assign it to a variable and use the variable instead. This will avoid indexing into the slice/string multiple times.
1. If there are multiple conditions in an `if` statement, try to move (or add) the non-indexing checks to the front of the statement. This sometimes will help short circuit the checks and avoid the slice-indexing checks.

One might question if this is worth optimizing, but like I said, I am trying to squeeze every oz of performance so 3.5% is still good for me. Unfornately I do know I won't get 3.5% since I can't remove every operation that index into slice/string.

**3. Reduce Heap Allocation**

This is true for all languages (where you can have some control of stack vs heap allocation), and it's even more true in Go. Mainly in Go, if you allocate a new slice, Go will "zero" out the allocated memory. This is great from a safety perspective, but it does add to the overhead. 

As an example, in the scanner I originally allocated a new _Sequence_ (slice of _Token_) for every new message. However, when i changed it to re-use the existing slice, the performance increased by over 10%!

```
  $ go run ./sequence.go bench scan -i ../../data/sshd.all
  Scanned 212897 messages in 0.87 secs, ~ 246027.12 msgs/sec

  $ go run ./sequence.go bench scan -i ../../data/sshd.all
  Scanned 212897 messages in 0.77 secs, ~ 275038.83 msgs/sec
```

The best thing to do is to run Go's builtin CPU profiler, and look at the numbers for Go internal functions such as `runtime.makeslice`, `runtime.markscan`, and `runtime.memclr`. Large percentages and numbers for these internal functions are dead giveaway that you are probably allocating too much stuff on the heap.

I religiously go through the SVGs generated from the Go profiler to help me identify hot spots where I can optimize.

Here's also a couple of tips I picked up from the [go-nuts mailing list](https://groups.google.com/forum/#!topic/golang-nuts/baU4PZFyBQQ):

* Maps are bad--even if they're storing integers or other non-pointer structs. The implementation appears to have lots of pointers inside which must be evaluated and followed during mark/sweep GC.  Using structures with pointers magnifies the expense.
* Slices are surprisingly bad (including strings and substrings of existing strings). A slice is a pointer to the backing array with a length and capacity. It would appear that the internal pointer that causes the trouble because GC wants to inspect it.

**4. Reduce Data Copying**

Data copying is expensive. It means the run time has to allocate new space and copy the data over. It's even more expensive when you can't have do `memcpy` of a slice in Go like you can in C. Again, direct memory copying is not Go's design goal. It is also much safer if you can prevent users from playing with memory directly too much. However, it is still a good idea to avoid any copying of data, whether it's string or slice.

As much as I can, I try to do in place processing of the data. Every _Sequence_ is worked on locally and I try not to copy _Sequence_ or string unless I absolutely have to. 

Unfortunately I don't have any comparison numbers for this one, because I learned from [previous projects](http://zhen.org/blog/surgemq-mqtt-message-queue-750k-mps/) that I should avoid copying as much as possible. 

**5. Mind the Data Struture**

If there's one thing I learned over the past year, is to use the right data structure for the right job. I've written about other data structures such as [ring buffer](http://zhen.org/blog/ring-buffer-variable-length-low-latency-disruptor-style/), [bloom filters](http://zhen.org/blog/benchmarking-bloom-filters-and-hash-functions-in-go/), and [skiplist](http://zhen.org/blog/go-skiplist/) before. 

However, [finite-state automata or machine](http://en.wikipedia.org/wiki/Finite-state_machine) is my latest love and I've been using it at various projects such as my [porter2](http://zhen.org/blog/generating-porter2-fsm-for-fun-and-performance/) and [effective TLD](https://github.com/surge/xparse/tree/master/etld). Ok, technical FSM itself is not a data structure and can be implemented in different ways. In the `sequence` project, I used both a tree representation as well as a bunch of switch-case statements. For the [porter2](http://zhen.org/blog/generating-porter2-fsm-for-fun-and-performance/) FSMs, I used switch-case to implement them.

Interestingly, swtich-case doesn't always win. I tested the time FSM using both tree and switch-case implementations, and the tree actually won out. (Below, 1 is tree, 2 is switch-case.) So guess which one is checked in?

```
BenchmarkTimeStep1   2000000         696 ns/op
BenchmarkTimeStep2   2000000         772 ns/op
```

Writing this actually reminds me that in the parser, I am currently using a tree to parse the sequences. While parsing, there could be multiple paths that the sequence will match. Currently I walk all the matched paths fully, before choosing one that has the highest score. What I should do is to do a weighted walk, and always walk the highest score nodes first. If at the end I get a perfect score, I can just return that path and not have to walk the other paths. (Note to self, more parser optimization to do).

**6. Avoid Interfaces If Possible**

This is probably not a great advice to give to Go developers. Interface is probably one of the best Go features and everyone should learn to use it. However, if you want high performane, avoid interfaces as it provides additional layers of indirection. I don't have performance numbers for the `sequence` project since I tried to avoid interfaces in high performance areas from the start. However, previous in the [ring buffer](http://zhen.org/blog/ring-buffer-variable-length-low-latency-disruptor-style/) project, the version that uses interface is 140% slower than the version that didn't.

I don't have the direct link but someone on the go-nuts mailing list also said:

>If you really want high performance, I would suggest avoiding interfaces and, in general, function calls like the plague, since they are quite expensive in Go (compared to C). We have implemented basically the same for our internal web framework (to be released some day) and we're almost 4x faster than encoding/json without doing too much optimization. I'm sure we could make this even faster.

**7. Find Ways to Short Circuit Checks**

Find ways to quickly eliminate the need to run a section of the code has been tremendously helpful to improve performance. For example, here are a couple of place where I tried to do that.

In [this first example](https://github.com/strace/sequence/blob/master/scanner.go#L223), I simply added `l == 1` before the actual equality check of the string values. The first output is before the add, the second is after. The difference is about 2% performance increase.

```
  $ go run ./sequence.go bench scan -i ../../data/sshd.all
  Scanned 212897 messages in 0.78 secs, ~ 272303.79 msgs/sec

  $ go run ./sequence.go bench scan -i ../../data/sshd.all
  Scanned 212897 messages in 0.76 secs, ~ 278433.34 msgs/sec
```

In [the second example](https://github.com/strace/sequence/blob/master/scanner.go#L282), I added a quick check to make sure the remaining string is at least as long as the shortest time format. If there's not enough characters, then don't run the time FSM. The performance difference is about 2.5%.

```
  $ go run ./sequence.go bench scan -i ../../data/sshd.all
  Scanned 212897 messages in 0.78 secs, ~ 272059.04 msgs/sec

  $ go run ./sequence.go bench scan -i ../../data/sshd.all
  Scanned 212897 messages in 0.76 secs, ~ 279388.47 msgs/sec
```

So by simply adding a couple of checks, I've increased perfromance by close to 5%.

### Conclusion

At this point I think I have squeezed every bit of performance out of the scanner, to the extend of my knowledge. It's performing relatively well and it's given the parser plenty of head room to do other things. I hope some of these lessons are helpful to whatever you are doing. 

Feel free to take a look at the [sequence](https://github.com/strace/sequence) project and try it out if you. If you have any issues/comments, please don't hestiate to [open a github issue](https://github.com/strace/sequence/issues).