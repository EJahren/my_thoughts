---
title: Mutation testing
---

> Never trust a test you haven't seen fail
>
> Marit van Dijk
> “Use Testing to Develop Better Software Faster”
> medium.com/97-things/use-testing-to-develop-better-software-faster-9dd2616543d3

Recently I had another chance to use the mutation testing package [mutmut](https://mutmut.readthedocs.io/en/latest/).
Mutation testing isn't a tool I have reached for often, but it has been fun although
a bit frustrating every time.

<!--more-->

The idea of mutation testing somehow makes many roll their eyes, thinking it
is akin to "chasing coverage". I would argue that mutation testing is a bit
less prone to false security than coverage and there really are things you
would like to test thoroughly before anything bad can happen (
[pacemaker firmware](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC6453543/)
and
[flight stabilizing software](https://www.theregister.com/2020/04/09/boeing_737_max_software_fixes/)
come to mind). So given that you want tests, what is it mutation
testing can do for you?

There are a number of questions that it could help to answer, but what the
mutation testing tools are primarily going to answer is "What kind of failures
did I forget to test for?". Its in no way a magic oracle that is going to give
you that answer perfectly, but I find it gives a good starting point.

The following examples will be using python, pytest and mutmut, but equivalent
tools exist for other languages.

So lets say you have figured out you need to make a change to `dispatch.py`
file in the `app.serialize` module, and the tests are located in the `tests/`
directory. Now you might want to know, which test will exercise code in this
file and what kind of mistakes will it stop you from doing? After putting the
following in a file named `.coveragerc`:

```
[html]
show_contexts = True
```

You can get the list of tests that run this file with

```bash
pytest tests/ --cov=app.serialize --cov-report=html --cov-context=test
```

This generates a nice little webpage in a directory named `htmlcov/` where each
line is annotated with which tests ran it. This gives you some baseline insight
about which test runs what. Which is already useful information to have before
you start changing the code in that file. mutmut can [be
configured](https://mutmut.readthedocs.io/en/latest/#selection-based-on-coverage-contexts)
to use this information to only run the relevant subset of tests for any
mutation. That really comes in handy as it otherwise runs all tests multiple
times for each line of code by default (time to go grab a coffee on the other
side of town).

What mutation testing does is to test your tests: introduce various mutants of
your program and see if the tests fail (kills the mutant). The mutmut tool
automates the process of generating mutants and running your tests. Those
mutations may be a buggy version of the code, as intended, or not, often called
a benign mutant. Benign mutants are therefore false positives. Whenever tests
pass for a particular mutant, manual inspection is necessary to determine
whether the test suite has a blind spot, the mutant is benign, or the failure
is not worth testing for.

As alluded to, there is significant overlap between this technique and coverage
analysis. Clearly, any mutation of a line not covered by tests will not be
detected by the tests. However, mutation testing gives you additional insight
about whether the test is merely running the code or whether it is asserting
something about correct behavior.

mutmut is configured in the `setup.cfg` file by adding the following:

```
[mutmut]
paths_to_mutate=src/app/serialize/dispatch.py
tests_dir=tests/
```

The command `mutmut run` will start running `mutmut`. You then
get the following progress screen which is pretty self-explanatory.

```
- Mutation testing starting -

These are the steps:
1. A full test suite run will be made to make sure we
   can run the tests successfully and we know how long
   it takes (to detect infinite loops for example)
2. Mutants will be generated and checked

Results are stored in .mutmut-cache.
Print found mutants with `mutmut results`.

Legend for output:
🎉 Killed mutants.   The goal is for everything to end up in this bucket.
⏰ Timeout.          Test suite took 10 times as long as the baseline so were killed.
🤔 Suspicious.       Tests took a long time, but not long enough to be fatal.
🙁 Survived.         This means your tests need to be expanded.
🔇 Skipped.          Skipped.

mutmut cache is out of date, clearing it...
1. Running tests without mutations
⠋ Running...Done

2. Checking mutants
⠇ 1/4  🎉 0  ⏰ 0  🤔 0  🙁 1  🔇 0
```

Once it finishes you get a summary of mutant ids that survived, e.g. 1-10,
and we can have a look at for instance the mutant with id 4 like this: `mutmut show 4`.
That output could look like this:

```diff
- accum = 0
+ accum = 1
```

The mutant changes the assigned value from `0` to `1` and the tests never caught on.
This could actually be perfectly fine and shouldn't matter to the tests. But
if this is actually a catastrophic problem, it may be a good idea to investigate
whether there is some nice test one could add which would make this fail.

If you decide to add some more tests you can rerun just this mutant with `mutmut run 4`.
