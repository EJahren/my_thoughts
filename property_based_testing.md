---
title: You should know property based testing
---

I was lucky enough to present at [[https://ndctechtown.com/agenda/omg-how-do-i-write-software-that-isnt-a-ticking-timebomb-09u2/0n60scjenit|NDC tech town]]
this year, and I had a great time! I spoke about how our team uses Property
Based Testing (PBT) to find bugs before they create problems. I got several
interesting questions which I answered fairly ok. However, in hindsight I could
have done better. So here is an attempt to summarize what the presentation was
about and answer those questions that came up.

First, what is PBT? I like to think of it as fuzzing for unit tests.

* scary bugs lists
  - Sometimes just incovenient and embarrasing https://www.smh.com.au/sport/olympic-fail-hammer-thrower-wrongly-celebrates-bronze-20120812-2420s.html
  - Sometimes ruining the lives of many: https://en.wikipedia.org/wiki/British_Post_Office_scandal

   
* The macintosh monkey
* fuzzing in high criticality oss: [oss-fuzz](https://github.com/google/oss-fuzz)
* link to infoq article
* video: https://youtu.be/fNZJZ6tg2Jo?t=9950
