+++
date = "2015-02-28T18:48:24-08:00"
title = "Patterns"
weight = 90
series_weight = 40
series = [ "library" ]
+++

<a href="#" class="image fit"><img src="/images/pic08.jpg" alt="" /></a>

The `sequence` _parser_ does not use regular expression. In fact, it won't understand any regular expression in the patterns even if you put them there. What it does recognize is a sequential pattern that follows the same format as the message it self. For fields that the analyst wants to extract, a field token of the form %fieldname% is put in its place.

As an example, this is pattern that's applied to the corresponding sudo message:


```
jan 15 14:07:04 testserver sudo: pam_unix(sudo:auth): password failed

%msgtime% %apphost% %appname% : pam_unix ( sudo : %action% ) : %method% %status%
```

In this example, we are tagging six different tokens of the message with semantic fields, including:

| Field | Value |
|-------|-------|
| %msgtime% | jan 15 14:07:04 |
| %apphost% | testserver |
| %appname% | sudo |
| %action% | auth |
| %method% | password |
| %status% | failed |

Note that it is not required to add spaces before and after the parenthesis. The parser uses the same scanner for message patters as it uses for the messages themselves, so the parenthesis will be extracted as separate tokens in both cases. It just seems to be more clear to have the spaces.

Here's another longer example (which you may have to horizontall scroll to see):

```
id=firewall time="2005-03-18 14:01:46" fw=TOPSEC priv=6 recorder=kernel type=conn policy=414 proto=TCP rule=accept src=61.167.71.244 sport=35223 dst=210.82.119.211 dport=25 duration=27 inpkt=37 outpkt=39 sent=1770 rcvd=20926 smac=00:04:c1:8b:d8:82 dmac=00:0b:5f:b2:1d:80

id = %appname% time = " %msgtime% " fw = %apphost% priv = %integer% recorder = %string% type = %string% policy = %policyid% proto = %protocol% rule = %status% src = %srcip% sport = %srcport% dst = %dstip% dport = %dstport% duration = %integer% inpkt = %pktsrecv% outpkt = %pktssent% sent = %bytessent% rcvd = %bytesrecv% smac = %srcmac% dmac = %dstmac%
```

### Field Tokens

A field token is of the format "%field:type:meta%":

* field is the name of the field
* type is the token type of the field
* meta is one of the following meta characters -, +, *, where
  * "-" means the rest of the tokens
  * "+" means one or more of this token
  * "*" means zero or more of this token

A field token can take different formats. The supported formats are:

* %field%
* %type%
* %field:type%
* %field:meta%
* %type:meta%
* %field:type:meta%
* %field:-:until%

#### Using the `-` Meta Character

Below is an example of how to use the `-` meta character:

```
jan 14 10:15:56 testserver sudo:    gonner : tty=pts/3 ; pwd=/home/gonner ; user=root ; command=/bin/su - ustream

%msgtime% %apphost% %appname% : %srcuser% : tty = %string% ; pwd = %string% ; user = %dstuser% ; command = %method:-%
```

In this example, most field tokens are specified in the %field% format, except for the last one. The last token, `%method:-`, specifies that the `%method%` token should consume the rest of the tokens. This means the `%method%` token will have the value of `/bin/su - ustream`. This field token can also be specified as `%method::-`.

#### Using the `+` Meta Character

Here's an example of using the `+` meta character:

```
Feb  8 21:51:10 mail postfix/pipe[84059]: 440682230: to=<userB@company.office>, orig_to=<userB@company.biz>, relay=dovecot, delay=0.9, delays=0.87/0/0/0.03, dsn=2.0.0, status=sent (delivered via dovecot service)

%msgtime% %apphost% %appname% [ %sessionid% ] : %msgid:integer% : to = < %srcemail% > , orig_to = < %string% > , relay = %string% , delay = %float% , delays = %string% , dsn = %string% , status = %status% ( %reason::+% )
```

In this example, the 2nd to the last token is `%reason::+%`. This means the `%reason%` field token will consume one or more tokens until the close parenthesis. This can also be written as `%reason:+%`.

#### Using the `*` Meta Character

Here's an example of using the `*` meta character:

```
id=firewall time="2005-03-18 14:01:46" fw=TOPSEC priv= recorder=kernel type=conn policy=414 proto=TCP rule=accept src=61.167.71.244 sport=35223 dst=210.82.119.211 dport=25 duration=27 inpkt=37 outpkt=39 sent=1770 rcvd=20926 smac=00:04:c1:8b:d8:82 dmac=00:0b:5f:b2:1d:80

id = %appname% time = " %msgtime% " fw = %apphost% priv = %integer:*% recorder = %string% type = %string% policy = %policyid% proto = %protocol% rule = %status% src = %srcip% sport = %srcport% dst = %dstip% dport = %dstport% duration = %integer% inpkt = %pktsrecv% outpkt = %pktssent% sent = %bytessent% rcvd = %bytesrecv% smac = %srcmac% dmac = %dstmac%
```

In this example, part of the pattern is specified as `priv = %integer:*%`. This means the value for `priv` may or may not be there. This can also be written as `%:integer:*%`.

#### Using the 'until' Format

A common use case is to consume all tokens until a specific token is hit. As an example, below is an example of a message that has some JSON-like text in the middle:

```
2015-01-24T19:34:47.269-0500 [conn72800] query foo.bar query: { _id: { $gte: { ContactId: BinData(3, 6C764EA2DABCE241C3E) }, $lt: { ContactId: BinData(3, 6C764EA2DAB4D9B1C3F) } } } planSummary: IXSCAN { _id: 1 } ntoreturn:0 ntoskip:0 nscanned:12 nscannedObjects:12 keyUpdates:0 numYields:10 locks(micros) r:2733 nreturned:12 reslen:4726 102ms
```

In this example, we want to extract anything after "query:" and before "planSummary" into the _object_ field. To do that, we can write the following rule:

```
%msgtime% [ %threadid% ] query %string% query : %object:-:plansummary% plansummary : %object::-%
```

Notice the token `%object:-:plansummary%`. What this says is that consume all the tokens from this point on, until the token `plansummary` is found. So the end result is `object` field will contain

```
{ _id: { $gte: { ContactId: BinData(3, 6C764EA2DABCE241C3E) }, $lt: { ContactId: BinData(3, 6C764EA2DAB4D9B1C3F) } } }
```

After this token is done, the rest of the rule continues. Because this token does not consume the token `plansummary`, we start the rule again from that.