+++
date = "2015-02-28T18:48:24-08:00"
title = "Token Types"
weight = 120
series_weight = 70
series = [ "library" ]
+++

The `sequence` _scanner_ will attempt to automatically recognize the following token types. 

| Type | Description |
|------|-------------|
| time | timestamp, in the format listed in [TimeFormats](/manual/timeformats/) |
| ipv4 | IPv4 address, in the form of a.b.c.d |
| ipv6 | IPv6 address |
| integer | integer number |
| float | floating point number of the form xx.yy |
| uri | URL, in the form of http://... or https://... |
| mac | mac address, in the form of aa:bb:cc:dd:ee:ff |
| string | string that reprensents multiple possible values |
| literal | a literal string, fixed value, not used by rules |