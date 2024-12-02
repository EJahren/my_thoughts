---
title: Using Model Based PBT to FUZZ a Web-API
---

I was lucky enough to present at
[[https://ndctechtown.com/agenda/omg-how-do-i-write-software-that-isnt-a-ticking-timebomb-09u2/0n60scjenit|NDC
tech town]] this year, and I had a great time! I spoke about how our team uses
Property Based Testing (PBT) to find bugs before they create problems. While
holding it I realised that the example of using stateful PBT
(sometimes called model based testing) for testing a web api, probably worked better
as a blog post than in a talk. So that is the topic of this blog post.

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

So how does stateful (or model based) PBT work? Say you have some system with
state, which can be interacted with in several ways. The classical example is a
database (the state), which can be queried, updated, added to, and deleted from
(the actions) which could change the state. In order to test this system we want
to select some valid actions, perform them and check that we get the correct behavior.
In order to know what behavior is correct, we keep a simplified model of how we
expect the real database to behave, for instance a hash map.

In order to start testing a web api with stateful PBT, we need to know the basics,
luckily there are several good introductory examples you can follow:

* [Database example from hypothesis documentation](https://hypothesis.readthedocs.io/en/latest/stateful.html)
* [Heap example by David R. MacIver](https://hypothesis.works/articles/rule-based-stateful-testing/)

So which web api could we use as an example? Let's start with [The flask
tutorial, named flaskr](https://github.com/pallets/flask/tree/a9b99b3489cc24b5bed99fbadbd91f598a5a1420/examples/tutorial).
This is a multi-user blogging application. What we are going to use as a model,
is overly simple, there is a set of registered users, one of which we are logged in as. Why
doesn't the model say anything about blog posts? Well, it could do so, but
what I find is the trick with stateful testing is to start with a small model and
then gradually refine it. Try not to have an overly complicated model though, as this
decreases the confidence in correctness.

The following is how we set up the test

```python
from flaskr import create_app
from flaskr.db import init_db

import tempfile
import os

import hypothesis.strategies as st
from hypothesis import assume
from hypothesis.stateful import RuleBasedStateMachine, rule, precondition

class StatefulFlaskrTest(RuleBasedStateMachine):
    def __init__(self):
        super().__init__()

        # start application
        self.db_fd, self.db_path = tempfile.mkstemp()
        self.app = create_app({"TESTING": True, "DATABASE": self.db_path})
        with self.app.app_context():
            init_db()
        self.client = self.app.test_client()

        # set up model
        self.registered = {}
        self.logged_in = None


    def teardown(self):
        # clean up application
        os.close(self.db_fd)
        os.unlink(self.db_path)
```

The setup and teardown of the application is described in the flask tutorial.
The `registered` dictionary holds the user names and the corresponding
password. `self.logged_in` is `None` when no user is logged in, otherwise it
is the name of the logged in user.

The first rule we create is the one that registered a randomly generated user:

```python
    @precondition(lambda self: self.logged_in is None)
    @rule(username=st.text(min_size=1), password=st.text(min_size=1))
    def register(self, username, password):
        assume(username not in self.registered)

        response = self.client.post(
            "/auth/register",
            data={
                "username": username,
                "password": password
            }
        )

        assert response.status_code == 302
        assert response.headers["Location"] == "/auth/login"

        # update model
        self.registered[username] = password
```

The precondition just says that we are currently not logged in.
`assume(username not in self.registered)` filters out user names that are already
registered. Then we perform the register action with the client and assert that
we get redirected to the login page. Finally our model is updated.

The next rule is to log in, but in order to do so we need to randomly draw a
registered user. The easiest way to do that is to use the `data` strategy:

```python
    def registered_users(self, draw):
        assume(self.registered)
        index = draw(st.integers())
        return list(self.registered.items())[index % len(self.registered)]

    @precondition(lambda self: self.logged_in is None)
    @rule(data=st.data())
    def log_in(self, data):
        username, password = self.registered_users(data.draw)

        response = self.client.post(
            "/auth/login", data={"username": username, "password": password}
        )

        assert response.status_code == 302
        assert response.headers["Location"] == "/"

        #update model
        self.logged_in = username
```

The `registered_users` function is responsible for generating a random registered user
given a function that can draw values from strategies, which we supply with the
`data.draw` function.

Again we have a precondition for the rule saying that we are not currently logged in. After
logging in with the client we assert that we get redirected to the posts page and finally we
update the model.

Logging out and posting follow very similar patterns:

```python
    @precondition(lambda self: self.logged_in is not None)
    @rule()
    def log_out(self):
        response = self.client.get("/auth/logout")

        assert response.status_code == 302
        assert response.headers["Location"] == "/"

        self.logged_in = None

    @rule(title=st.text(), body=st.text())
    def create(self, title, body):
        response = self.client.post(
            "/create",
            data={
                "title": title,
                "body": body
            }
        )

        if self.logged_in is None:
            assert response.status_code == 302
            assert response.headers["Location"] == "/auth/login"
        else:
            response.status_code == 200

```

Note that if we are logged in we assert that we get status code 200 and redirected to
the login page otherwise.

Like I said to begin with, our model is overly simple, so how could we improve it?

* Multiple logged in users, which would mean `self.logged_in` becomes a collection of
    logged in users and their session.
* Keeping track of created blog posts, another field `self.posts` with the created
    blog posts and checking that they can be viewed.

Which I leave as an exercise for the reader.
