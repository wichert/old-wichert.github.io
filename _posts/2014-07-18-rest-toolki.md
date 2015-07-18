---
title: Get your REST on
description: |
  TL;DR: [rest_toolkit](https://rest-toolkit.readthedocs.org/) is a new
  framework to create REST applications, build using Pyramid.

  For two recent projects involved implementing a REST service. The experience
  of doing that led me to create a new simple but extendible toolkit for buidling
  REST applications. Want to see how simple? This is a full REST application:

  ```python
  @resource('/')
  class Greeting(object):
      def __init__(self, request):
          pass

  @Greeting.GET()
  def show_root(root, request):
      return {'message': 'Hello, world'}

  quick_serve()
  ```

---
TL;DR: [rest_toolkit](https://rest-toolkit.readthedocs.org/) is a new framework to
create REST applications, build using Pyramid.

In two recent projects I needed to build a REST services which would be used
with mobile clients. When I started the first project I looked at
[cornice](http://cornice.readthedocs.org/en/latest/), a fairly popular REST
framework for pyramid. For various reasons its design did not appeal to me, so
I decided to just use
[Pyramid](http://docs.pylonsproject.org/en/latest/docs/pyramid.html) directly,
without using a special REST layer. This worked very well and gave me a lot of
extra flexibility. While going through this process several patterns emerged,
so before I started the second project I extracted those to create a new
mini-framework for creating REST applications base. This resulting framework is
very tiny, and is based several design criteria:

* Defining REST resources must be as simple as possible.
* Authentication and authorization to control access must be fully supported.
* The common scenario of using SQLAlchemy models to define data
  must be as simple as possible.
* Using the REST framework most feel natural to both newcomers and people who
  are already (very) familiar with Pyramid.
* The REST framework must allow full use of the underlying Pyramid system..

The end result is [rest_toolkit](https://rest-toolkit.readthedocs.org/). It runs
on Python 2.7 and 3.3+ and is very easy to use.

rest_toolkit in action
-----------------------

Here is an example of a simple REST application using this framework:

```python
from rest_toolkit import quick_serve
from rest_toolkit import resource


@resource('/')
class Greeting(object):
    def __init__(self, request):
        pass


@Greeting.GET()
def show_root(root, request):
    return {'message': 'Hello, world'}


quick_serve()
```

This example defines a resource which can be accessed at the site root
and can handle GET requests. Accessing this with HTTP client gives the expected
result:

```
$ curl http://localhost:8080
{"message": "Hello, world"}
```

Using an HTTP request method for which no view is defined will return a HTTP
405 error:

```
$ curl -v  -X PUT http://localhost:8080
[...]
* HTTP 1.0, assume close after body
< HTTP/1.0 405 Method Not Allowed
< Date: Fri, 18 Jul 2014 14:59:40 GMT
< Server: WSGIServer/0.2 CPython/3.4.1
< Content-Type: application/json; charset=UTF-8
< Content-Length: 38
<
* Closing connection 0
{"message": "Unsupported HTTP method"}
```

An OPTIONS handler which lists all allowed HTTP methods is provided automatically:

```
$ curl -v -X OPTIONS http://localhost:8080
[...]
* HTTP 1.0, assume close after body
< HTTP/1.0 204 No Content
< Date: Fri, 18 Jul 2014 14:35:07 GMT
< Server: WSGIServer/0.2 CPython/3.4.1
< Access-Control-Allow-Methods: GET, OPTIONS
< Content-Length: 0
<
* Closing connection 0
```


Default views
-------------

Writing views for create, read, update and delete (CRUD) actions for every
resource quickly becomes vary tedious. rest_toolkit solves this by providing
default views. A resource can opt in to these views through a set of abstract
base classes.

```python
from rest_toolkit import resource
from rest_toolkit.abc import ViewableResource


@resource('/balloon/{id}')
class BalloonFigure(ViewableResource):
    def __init__(self, request):
        ...

    def to_dict(self):
        # Return a dictionary with resource information, which will be used
        # by GET reqests
        return {'id': self.id,
                'figure': self.figure,
                'colour': self.colour}
```

The above example uses `ViewableResource`, which tells rest_toolkit that it should
handle GET requests for this resource. The `to_dict()` method will be used to generate
the data for the response.

Updating a resource
-------------------

REST resources can be updated using the PATCH and PUT HTTP methods. rest_toolkit
can handle that automatically via the EditableResource base class. This requires
a resource to implement two methods: `validate()` to validate the new data,
`update_from_dict()` to update a resource using the provided data, and `to_dict()`
to generate the response data. The most complex part of this is likely to be validating
the new data. Luckily this is a problem that is already handled by standard form
toolkits. rest_toolkit includes mix-in classes with a `validate()`-implementation
using either JSON schemas or [colander](https://colander.readthedocs.org/). Building
on our balloon figure example we use this to add PATCH/PUT support.

```python
from rest_toolkit import resource
from rest_toolkit.abc import EditableResource
from rest_toolkit.ext.jsonschema import JsonSchemaValidationMixin


@resource('/balloon/{id}')
class BalloonFigure(EditableResource, ViewableResource, JsonSchemaValidationMixin):
    schema = {
            '$schema': 'http://json-schema.org/draft-04/schema',
            'type': 'object',
            'properties': {
                'figure': {
                    'type': 'string',
                 },
                 'colour'': {
                     'type': 'string',
                     'choice': ['blue', 'green', 'ref', 'yellow'],
                 },
             },
             'additionalProperties': False,
             'required': ['figure', 'colour'],
     }

     def update_from_dict(self, data, replace):
         # This method must update the resource data.
```


Using data from SQL
-------------------

It is not uncommon to use data stored in a SQL database. rest_toolkit includes
a SQL extension which makes it very easy to expose your SQL data through REST
interface. By building on [SQLAlchemy](http://www.sqlalchemy.org/) and
[pyramid_sqlalchemy](https://pyramid-sqlalchemy.readthedocs.org) you only need
to define a resource with a SQL query to find the relevant data in a database.
Here is an example for a database that lists possible balloon figures.


```python
from wsgiref.simple_server import make_server
from pyramid.config import Configurator
from pyramid_sqlalchemy import BaseObject
from sqlalchemy import bindparam, schema, types
from sqlalchemy.orm import Query
from rest_toolkit import resource
from rest_toolkit.abc import ViewableResource
from rest_toolkit.ext.sql import SQLResource


class BalloonFigure(BaseObject):
    __tablename__ = 'balloon'

    id = schema.Column(types.Integer(), primary_key=True, autoincrement=True)
    figure = schema.Column(types.Unicode(), nullable=False)
    colour = schema.Column(types.Unicode(), nullable=False)


@resource('/balloons/{id}')
class BalloonFigureResource(SQLResource, ViewableResource):
    context_query = Query(BalloonFigure).filter(BalloonFigure.id == bindparam('id'))


config = Configurator(settings={'sqlalchemy.url': 'postgresql:///circus'})
config.include('rest_toolkit')
config.include('pyramid_sqlalchemy')
config.scan()
app = config.make_wsgi_app()
server = make_server('0.0.0.0', 5000, app)
server.serve_forever()
```

After creating the database and inserting some data you can query the
REST:

```
$ curl http://localhost:5000/balloons/1
{"figure": "Giraffe", "id": 1, "colour": "Yellow"}
```

Two things happened here:

1. The object id was extracted from the URL using the ``/balloons/{id}`` route
   path, and then inserted in the SQL query to find the right object.
2. The BalloonFigure class was inspected to find all available attributes, which
   were used to generate the response.


More to come
------------

rest_toolkit is very new, but it is already a very useful tool to easily create
REST applications. There is certainly still a room for further improvements and
better documentation. If you want to contribute please check out the
[github project](https://github.com/wichert/rest_toolkit).
