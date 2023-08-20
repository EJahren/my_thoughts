# Mutation testing: the superpower you always had but didn't know about

# THIS IS A DRAFT!

When do you know whether you have added sufficient amount of
test cases? Perhaps you use TDD and stop once the feature,
is completed, but then are you really sure that those are all
the cases? Also, there is no lasting proof that TDD was
applied, so how can your coworkers verify that there are arr sufficient
tests? Perhaps you aim for high line coverage? Indeed, uncovered code nearly
always signals missing test cases, but 100% coverage does not mean no missing
test cases.

I have found, again and again, that [mutation testing](https://en.wikipedia.org/wiki/Mutation_testing) is an invaluable tool to uncover gaps in test coverage (which often leads to discovering bugs),
and faulty error handling. You can use tools such as [mutmut](pypi.org/mutmut]
which apply the technique automatically using a fixed set of patterns
(change +1 to -1, mangle strings, etc.) to create "mutants" from, but
performing the steps manually means you can
be more creative with your mutants. In essence, mutation testing is just
the pragmatic application of the following advice:


> Never trust a test you haven't seen fail
>https://youtu.be/kTcDBYCpj7Q
> Marit van Dijk
> “Use Testing to Develop Better Software Faster”
> medium.com/97-things/use-testing-to-develop-better-software-faster-9dd2616543d3

This analysis has been given many names, G. M. Weinberg called it
"bebugging" in "The pshychology of Computer Programming" (1970). In more
general terms we may call it [fault injection](https://en.wikipedia.org/wiki/Fault_injection).
Simply change the program slightly to a buggy mutant program and see if the tests fail. Mutants
it passes the tests.

So let's take a truly toy example borrowed from [Kevlin
Henney's "structure and interpretation of test
cases"](https://www.youtube.com/watch?v=MWsk1h8pv2Q&t=892s). Let's implement
is_leap_year in python.

## leap year mutants

As far as I am concerned, there is only one correct way of doing this
in python:


```python
from calendar import isleap
```

But for the sake of argument, let's have a specification:

> To be a leap year, the year number must be divisible by four – except for
> years divisble by 100, unless they are also be divisible by 400.

The first part of this specification translates very easily into a test:

```python
import pytest

@pytest.mark.parametrize("year", [1904, 1914, 1918, 1939, 1945, 1908, 2004])
def test_that_only_years_divisible_by_four_are_leap_years(year):
    if year % 4 != 0:
        assert not is_leap_year(year)
```

Now this test has a bit of logic in it, that isn't really necessary. Many would
consider this a bit of a code smell, and I agree. However, it is the direct
translation of the specification. Also, writing the test in this way makes it
very easy to translate into a property test using
[hypothesis](hypothesis.works) so you get to have a large amount of test cases
basically for free, which I am a big fan of. Hopefully, the logic doesn't
hide anything nefarious from us...

Now, lets err on the side of following TDD too literally, and get going
with some implementation. Perhaps not everyone would agree at this point,
but later we will have some fun, I promise!

```python
def is_leap_year(year: int) -> bool:
    return year % 4 == 0
```

Clearly flawed, and if you have ever done any programming with dates you
are probably sitting uncomfortably in your chair right now. So lets move swiftly on:

```python
@pytest.mark.parametrize("year", [1600, 1700, 1800, 1900])
def test_that_years_divisible_by_100_are_not_leap_years_unless_divisible_by_400(year):
    if year % 100 == 0:
        assert not is_leap_year(year) or year % 400 == 0

```

which brings us to the final implementation:

```python
def is_leap_year(year: int) -> bool:
    return (year % 4 == 0
      and year % 100 != 0
      or year % 400 == 0
    )
```

As already noted by Kevlin Henney, this isn't very readable for humans, but it
is really easy to create mutants of. There are three comparisons, two boolean
connectives, 3 modulos and 6 integer constants. Each one of those can be
changed to create a mutant. So for instance `year % 4 == 0 and year % 231 != 0
or year % 400 == 0` is an obviously incorrect mutant, while `year % 4 == 0 or
year % 100 != 0 and year % 400 == 0` is more subtle. Luckily, both mutants are
killed by our tests, but how many mutants can we come up with that aren't
killed? Turns out the following implementation passes the tests:


```python
def is_leap_year(year: int) -> bool:
    return False
```

Now, admittedly I have played a bit of a trick. The tests have tried their best
to translate the requirements as written but have conveniently not observed any
of the return values directly. In the specification there is a case hidden by
natural language:


> Year numbers divisible by four and not divisible by 100 are leap years

So lets add that:

```python

@pytest.mark.parametrize("year", [1900, 1908, 1914, 1918, 2004])
def test_that_years_divisible_by_4_and_not_by_100_are_leap_years(year):
    if year % 4 == 0 and year % 100 != 0:
        assert is_leap_year(year)
```


## Why didn't my TDD discover the hidden test case?

Arguably, I made a mistake by not creating the following
function first:

```python
def is_leap_year(year: int) -> bool:
    return False
```

And then running the tests to check that they were still red.
To be honest, I genuinly did not realise that constant False
passed the test the first time I implemented that function.

I don't think anyone would suggest that we throw away the code
and start implementing from scratch. Fundamentally, we are not
trying to ensure that "good TDD practice was observed", what we
actually want out of the process is that:

* All of the code is designed to be testable
* Every requirement has a test and all code is needed for a requirement
* All tests can fail for some incorrect version of the code

These are certainly great goals and TDD is a really efficient way of getting
there. It's like buying the groceries on the way home from work; you save
yourself two trips. That being said, we really want it to do at least one more
thing:

* There should be enough test cases to avoid bugs

As has been repeated ad nauseum, testing cannot prove the absense of bugs,
but at least it shouldn't be too easy to imagine bugs that are not caught.
