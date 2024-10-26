---
title: You should know property based testing
---

I was lucky enough to present at [[https://ndctechtown.com/agenda/omg-how-do-i-write-software-that-isnt-a-ticking-timebomb-09u2/0n60scjenit|NDC tech town]]
this year, and I had a great time! I spoke about how our team uses Property
Based Testing (PBT) to find bugs before they create problems. I got several
interesting questions which I answered fairly ok. However, in hindsight I could
have done better. So here is an attempt to summarize what the presentation was
about and answer those questions that came up.

First, what is PBT? I like to think of it as fuzzing for unit tests. There
are several implementations for several languages (you can find a list [here](github.com/jmid/pbt-frameworks)!)
It usually consists of some library for creating generators for the data your
application uses and a way to inject that test data into a test method which
is ran several times. Here is an example using the PBT framework hypothesis for
python:

```python
from hypothesis import given
import hypothesis.strategies as st

@given(st.lists(elements=st.integers()))
def test_sorted(list):
    sorted_list = sorted(list)

    assert_is_permutation(list, sorted_list)
    assert_is_ordered(sorted_list)

def assert_is_permutation(list1, list2):
    for element in (list1+list2):
        assert list1.count(element) == list2.count(element)

def assert_is_ordered(list):
    for i in range(len(sorted)-1):
        assert list[i] <= list[i+1]
```

Here we have a generator for lists of integers:
`st.lists(elements=st.integers())` which is given to the test `test_sorted`
which sorts the list and asserts properties of sorting: that it returns an
ordered permutation of the input. The test will be run 100 times (by default)
with different lists.


* scary bugs lists
  - Sometimes just incovenient and embarrasing https://www.smh.com.au/sport/olympic-fail-hammer-thrower-wrongly-celebrates-bronze-20120812-2420s.html
  - Sometimes ruining the lives of many: https://en.wikipedia.org/wiki/British_Post_Office_scandal


* The macintosh monkey
* fuzzing in high criticality oss: [oss-fuzz](https://github.com/google/oss-fuzz)
* link to infoq article
* video: https://youtu.be/fNZJZ6tg2Jo?t=9950
