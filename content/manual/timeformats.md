+++
date = "2015-02-28T18:48:24-08:00"
title = "Time Formats"
+++

<a href="#" class="image fit"><img src="/images/pic06.jpg" alt="" /></a>

One of the most difficult task in log parsing is recognizing and understanding the different time formats. Syslog RFCs have defined a couple formats that the syslog servers abide to. However, there are many log messages out there that don't use the two syslog time formats. 

`sequence` currently automatically recognizes 42 different time formats as listed below. More will be added, so the best (authoritative) place to check is [sequence.toml](https://github.com/trustpath/sequence) in the github repo.

When writing rules, analysts don't need to know what time formats are used in the log messages. They just need to know that there's a time at a certain location of the message. `sequence` will automatically detecting the time format, and normalizing the time formats into a single one.

In the table below, the reference time used in the layouts is the specific time:

```
Mon Jan 2 15:04:05 MST 2006
```

which is Unix time 1136239445. Since MST is GMT-0700, the reference time can be thought of as

```
01/02 03:04:05PM '06 -0700
```

The best way to remember this is that the numbers are contiguous: 01, 02, 03, 04, 05, 06, 07 in the above format. Make sure you write the time format correctly as it will be used to normalize the different time stamps.

| # | Time Formats |
|---|--------------|
| 1. | Mon Jan _2 15:04:05 2006 |
| 2. | Mon Jan _2 15:04:05 MST 2006 |
| 3. | Mon Jan 02 15:04:05 -0700 2006 |
| 4. | 02 Jan 06 15:04 MST |
| 5. | 02 Jan 06 15:04 -0700 |
| 6. | Monday, 02-Jan-06 15:04:05 MST |
| 7. | Mon, 02 Jan 2006 15:04:05 MST |
| 8. | Mon, 02 Jan 2006 15:04:05 -0700 |
| 9. | 2006-01-02T15:04:05Z07:00 |
| 10. | 2006-01-02T15:04:05.999999999Z07:00 |
| 11. | Jan _2 15:04:05 |
| 12. | Jan _2 15:04:05.000 |
| 13. | Jan _2 15:04:05.000000 |
| 14. | Jan _2 15:04:05.000000000 |
| 15. | _2/Jan/2006:15:04:05 -0700 |
| 16. | Jan 2, 2006 3:04:05 PM |
| 17. | Jan 2 2006 15:04:05 |
| 18. | Jan 2 15:04:05 2006 |
| 19. | Jan 2 15:04:05 -0700 |
| 20. | 2006-01-02 15:04:05,000 -0700 |
| 21. | 2006-01-02 15:04:05 -0700 |
| 22. | 2006-01-02 15:04:05-0700 |
| 23. | 2006-01-02 15:04:05,000 |
| 24. | 2006-01-02 15:04:05 |
| 25. | 2006/01/02 15:04:05 |
| 26. | 06-01-02 15:04:05,000 -0700 |
| 27. | 06-01-02 15:04:05,000 |
| 28. | 06-01-02 15:04:05 |
| 29. | 06/01/02 15:04:05 |
| 30. | 15:04:05,000 |
| 31. | 1/2/2006 3:04:05 PM |
| 32. | 1/2/06 3:04:05.000 PM |
| 33. | 1/2/2006 15:04 |
| 34. | 02Jan2006 03:04:05 |
| 35. | Jan _2, 2006 3:04:05 PM |
| 36. | 2006-01-02T15:04:05Z |
| 37. | 2006-01-02T15:04:05-0700 |
| 38. | 2006-01-02T15:04:05.999-0700 |
| 39. | 2006-01-02 |
| 40. | 15:04:05 |
| 41. | 2006-01-02T15:04:05.999999Z |
| 42. | 02/Jan/2006:15:04:05.999 |
