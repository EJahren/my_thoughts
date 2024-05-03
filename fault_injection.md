# Mutation testing

> Never trust a test you haven't seen fail  
>
> Marit van Dijk  
> “Use Testing to Develop Better Software Faster”  
> medium.com/97-things/use-testing-to-develop-better-software-faster-9dd2616543d3  

Recently I dove back into mutation testing softare such as [mutmut](https://mutmut.readthedocs.io/en/latest/) and
it really opened my eyes to how powerful, yet simple, fault injection is as a technique. The
simple idea is to introduce some likely failure and observe the behavior of your system.

I recently became aware of a project where authentication was disabled by
mistake, meaning no credencials were necessary to access. There was tests for authentication,
but no test for "unauthorized users do not have access". Both creating the test and uncovering the
fault manually is quite simple.

Let say you have a `profile.html` page requiring sign in:

```python
@main.route('/profile')
@login_required
def profile():
    return render_template('profile.html')
```

Let's say we wonder if breaking authentication would trigger an alert. Then we could attempt to
run the tests with the following injected bug:

```python
@main.route('/profile')
# @login_required
def profile():
    return render_template('profile.html')
```

If the tests break then there is some protection against such a failure. However, this
is no simpler or less laborious than manually looking for such faults. It is however possible
to completely automate the fault injection process with e.g. [mutmut](https://mutmut.readthedocs.io/en/latest/).
That way, the lack of protection can be automatically flagged.
