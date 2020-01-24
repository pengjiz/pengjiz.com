---
title: Lessons Learned When Building an API Server
date: "2018-05-19T14:17:54-04:00"
keywords: ["dev", "python"]
description: Random thoughts on backend development.
draft: false
---

In the past two months, my major work was developing a parking reservation
service. For the project, I needed to design, write and deploy an API server, a
web app and an iOS app -- pretty intense, and a great fraction of the work was
out of my comfortable zone. For me, this was a wonderful journey, and I want to
share some of my thoughts I gained during the process. This post is on the
backend.

## My Mistakes with Python

For some reason the programming language for this project had to be Python. I
did use Python a lot mainly for machine learning, Bayesian statistics, and
sometimes small command line apps. However, I did not have any experience of
making real world applications in Python, and I did a lot of things wrong. Here
are the top three of my mistakes.

1. Insufficient testing. I do not like writing test cases. Usually I only test a
   few important functions, like the algorithm implementation part. That might
   be fine for data analysis of which the work is mostly interactive. However,
   when developing an API server, insufficient testing together with dynamic
   typing make refactoring very painful.

2. Dependency management. `conda` has been my tool of choice for managing python
   versions and virtual environments for a long time. For me, `conda` is better
   than `virtualenv` because the Python interpreter is also managed. Besides,
   when installing packages, even though now most Python packages provide
   wheels, `conda` still has the advantage of being able to install dependencies
   outside the Python world, which is pretty handy when using some scientific
   packages. However, for a web app, `conda` is not a suitable tool because many
   packages are not available. For this project, I chose to create virtual
   environment with `conda` but install packages with `pip`. I specified all the
   dependencies with exact version numbers in the `requirements.txt` file, and
   installed all the development dependencies with a `Makefile` target. This
   approach works, but is not very flexible and reliable. I have to upgrade
   dependencies manually, and the environment is not guaranteed to be exactly
   the same every time because I cannot freeze the dependencies (otherwise the
   development dependencies will also be locked). `pipenv` seems to be able to
   handle all those issues, but I need to find a way to use it with `conda`.

3. Iterators. Python iterators and generators can be quite useful, but they have
   bitten me several times. It is very easy to drain an iterator accidentally
   and then get some cryptic errors. I do agree that iterators should be
   "collected" if we want to save it for non-temporarily purposes, but it is
   very easy (at least for me) to forget. Because a lot of functions (`map`,
   `filter`, `zip`, etc) return iterators (in Python 3) and there is no way to
   notice it before our program gives wrong results. I think type annotations
   and static type checking will help, but the ecosystem is still not good now.

## Web Framework Choice

There are a few web frameworks in Python: Django, Flask, Tornado, etc. Before
this project, I once made a blog app with Django, and a dashboard with Flask.
Overall I prefer Flask to Django mainly because the former is more flexible, but
I did find a few points of Flask that I thought were not so good.

The first thing is the context locals. This is actually a pretty interesting
design of Flask and allows you to type less, but at the beginning this feature
was confusing. In Flask, there are a few objects that are [*local
proxies*][local proxies]. The value of such an object depends on the context.
For example, in Flask, if you want to get the request data, you would use:

```python
from flask import request

def view_fn():
    form = request.form
    # do something with form data
    pass
```

The `request` object is imported from the package. In Django (and most of other
frameworks), you would do:

```python
def view_fn(request):
    form = request.POST
    # do something with form data
    pass
```

The `request` object is an argument of the view function. Context locals do save
a bit typing, but I felt confused at the beginning.

The second one is that Flask tends to hide details. For instance, Flask provides
a function `abort` for terminating the request handling process:

```python
class Aborter(object):
    def __init__(self, mapping=None, extra=None):
        if mapping is None:
            mapping = default_exceptions
        self.mapping = dict(mapping)
        if extra is not None:
            self.mapping.update(extra)

    def __call__(self, code, *args, **kwargs):
        if not args and not kwargs and not isinstance(code, integer_types):
            raise HTTPException(response=code)
        if code not in self.mapping:
            raise LookupError('no exception for %r' % code)
        raise self.mapping[code](*args, **kwargs)


def abort(status, *args, **kwargs):
    return _aborter(status, *args, **kwargs)


_aborter = Aborter()
```

This function basically raises an exception. However, using this function hides
the fact of raising an exception and somewhat breaks the normal pattern in
Python. Such things are not a big deal after you become familiar with Flask, but
I do not like them.

The third thing is on extensions. The extensions of Flask are mostly
lightweight. That is good for me in most cases, but sometimes I feel very
jealous of those full-featured Django extensions. For example, as we all know
Django has a builtin `admin` module, which is very well designed, while the
[Flask-Admin] seems not very usable to me.

## API Design

In short, the lesson I learned was to use GraphQL if possible. It makes the
frontend part much easier, and even if you hate your frontend partner, it does
make the backend development less painful.

In this project, the API was still the good old REST API, mainly because the
Python GraphQL implementation [Graphene] was still in its infancy when I made
the decision. I chose [Flask-RESTful] and [marshmallow] for building the API.
For one database table, I needed to write at least two resources for common
operations, and for one resource, sometimes I needed to write three schemas for
data validation and serialization. Compared to [Prisma], that feels quite
verbose.

## Deployment

Actually deployment was the most time consuming part in the backend project, and
I learned a lot from it. The server we are using is one managed by CMU. It had
been used for hosting Django apps before so many libraries and packages were
available when I got it. Therefore, at first I decided to deploy the app
directly on the server, which I think is a very silly decision now. There were
so many weird issues to deal with -- outdated packages, dependency conflicts,
etc. Then I switched to Docker and things became a lot easier. I could upgrade
the app and dependencies freely -- just build and push a new image and run a new
container, and I could also use my only laptop as a staging machine without much
pain. I am still not very familiar with Docker but the experience so far is
positive.

[local proxies]: http://werkzeug.pocoo.org/docs/0.14/local/
[Flask-Admin]: https://github.com/flask-admin/flask-admin
[Graphene]: http://graphene-python.org/
[Flask-RESTful]: https://flask-restful.readthedocs.io/en/latest/
[marshmallow]: https://marshmallow.readthedocs.io/en/latest/
[Prisma]: https://www.prisma.io/
