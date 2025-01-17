.. minik documentation master file, created by
   sphinx-quickstart on Tue Feb 19 14:49:40 2019.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Minik - Serverless Web Framework
================================

|circle| |pypi version| |apache license|

Minik is a python microframework used to write clean APIs in the serverless domain.

As a developer working in AWS, if you would like to have full control of your development
process while having access to a familiar interface, minik is the framework for you.
To build your first API using minik, SAM and Juniper go to :doc:`quickstart`.

Familiar interface
******************

.. code:: python

    from minik.core import Minik
    app = Minik()

    @app.route("/test/{proxy}")
    def hello(proxy):
        return {"Hello": proxy}

HTTP Methods
************
With minik you can also specify the HTTP methods for a given view. If you don't
define the methods, every single HTTP method will be allowed by default.

.. code-block:: python

    from minik.core import Minik

    app = Minik()

    @app.route('/events/{location}')
    def events_view(location: str):
        # This route will be invoked for GET, POST, PUT, DELETE...
        return {'data': ['granfondo MD', 'Silver Spring Century']}

    @app.route('/events', methods=['POST', 'PUT'])
    def create_event_view():
        create_event(app.request.json_body)
        return {'result': 'complete'}

The microframework also includes a set of convenient decorator methods for the
case in which a view is associated with a single HTTP method.

.. code-block:: python

    from minik.core import Minik

    app = Minik()

    @app.get('/events/{location}')
    def get_view(location: str):
        return {'data': ['granfondo MD', 'Silver Spring Century']}

    @app.post('/events')
    def post_view():
        create_event(app.request.json_body)
        return {'result': 'complete'}


Route Validation
****************
Using the `function annotations`_, you can specify the type of value you are expecting
in your route. The added advantage is that minik will convert your parameter to the
appropriate type. For instance:

.. code:: python

    @app.route('/articles/{author}/{year}/')
    def get_articles_view(author: str, year: int):
        assert isinstance(author, str) and isinstance(year, int)
        return {'author_name': author, 'year': year}

If you need to specify a regular expression:

.. code-block:: python

    from minik.fields import ReStr

    @app.route('/item/{item_id}/', methods=['GET'])
    def get_item(item_id: ReStr(r'([0-9a-f]{8}$)')):
        assert isinstance(item_id, str)
        return {'id': item_id}

You can extend our validation framework and write your own classes for route fields!

.. code-block:: python

    from minik.fields import BaseRouteField

    class RouteTracker(BaseRouteField):

        def validate(self, value):
            return value in ('fitbit', 'nikeplus', 'vivosmart',)


    @app.route('/tracker/{name}/', methods=['GET'])
    def get_tracker_info(name: RouteTracker):
        assert isinstance(name, str)
        return {'name': name}

In the example above, your view will only be executed when the name paramter passed in
matches one of the trackers specified in the validator. All you need to do is implement the
validation logic. To learn more checkout out the `features`_ page.

.. _`function annotations`: https://www.python.org/dev/peps/pep-3107/
.. _`features`: https://eabglobal.github.io/minik/features.html


Custom Headers
**************
To update the values of the HTTP response, minik exposes a response object at
the app level. By default minik will create a Response instance with a status code
of 200 and a set of default headers. The headers include a default content-type
value of `application/json`.

For instance, to set the CORS headers in a view and change the content type, a
view would look like:

.. code:: python

    app = Minik()

    @app.get('/articles/{author}/{year}/')
    def get_articles_view(author: str, year: int):
        app.response.headers = {
            "Content-Type": "text/html; charset=utf-8",
            "Access-Control-Allow-Origin": "*",
            "Access-Control-Allow-Methods": "GET",
            "Access-Control-Allow-Headers": "Content-Type,X-Amz-Date",
            "Authorization": "X-Api-Key,X-Amz-Security-Token"
        }

        return f"A very short article by {author}"


Debug Mode
**********
For unhandled exceptions, minik will respond with a 500 status code and
a generic error message. To get more details from the response including the stack
trace and information about the exception, run the app in debug mode.

By default the debug mode is set to False.

.. code:: python

    app = Minik(debug=True)

Initializing the app in debug mode will relay the stack trace back to the consumer.


Motivation
**********
The team behind this framework is adopting a very minimal set of features to enhance
and streamline web development in the serverless space. These were the business
needs that encouraged us to build minik:

- Ability to write an API using a familiar (Flask like) syntax using serverless
  services.
- Flexibility on how to build and deploy lambda functions. I do not want
  my framework to dictate these processes for me. I want to own them!
- When installing a web framework, I want to get only the framework. I don’t
  want any additional tooling or any additional process-based workflows.
- When using the microframework I am responsible for the configuration
  required to associate my lambda function to its endpoints.


The features of this library should be absolutely driven by a very specific
business need. So far, the minimal approach has been sufficient for our team to
write and expose an API using AWS services.

Just the framework
******************
Things to be aware of when working with minik:

- When used in your lambda function, you're responsible for including the source
  code of minik in your .zip artifact. For packaging purposes we recommend using
  `Juniper`_.
- Unlike other frameworks like Flask or Django, where using the decorator is
  sufficient to define the routes of the web app, in minik, you’re responsible
  for linking a lambda function to the API gateway. We recommend using a
  `SAM`_ template.
- Minik does not include a local development server! For testing purposes, you can
  either deploy your lambda to AWS using `sam package` and `sam deploy`. For local
  deployment purposes you can use `sam local`.


Minik in λ
**********
When working with a lambda function as the handler of a request from the API gateway.
If the endpoint is configured to use the pass-through lambda integration, your function
will have to be defined as follows:

.. code:: python

    def lambda_handler(event, context):
        # Business logic! Get values out of the event object.
        return {
            'statusCode': 200,
            'headers': 'response headers',
            'body': 'response body'
        }

In this approach, the event your lambda function receives from the gateway looks
like this:

.. code:: json

    {
        "path": "/test/hello",
        "headers": {
            "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8",
            "Accept-Encoding": "gzip, deflate, lzma, sdch, br",
            "..."
        },
        "pathParameters": {
            "proxy": "hello"
        },
        "requestContext": {
            "accountId": "123456789012",
            "resourceId": "us4z18",
            "stage": "test",
            "requestId": "41b45ea3-70b5-11e6-b7bd-69b5aaebc7d9",
            "..."
            "resourcePath": "/{proxy+}",
            "httpMethod": "GET",
            "apiId": "wt6mne2s9k"
        },
        "resource": "/{proxy+}",
        "httpMethod": "GET",
        "queryStringParameters": {
            "name": "me"
        },
        "stageVariables": {
            "stageVarName": "stageVarValue"
        }
    }

Without Minik, every single API endpoint you write will need to parse that object
as a way to get the data values you care about. The entire scope of the object is
documented `here <https://docs.aws.amazon.com/lambda/latest/dg/eventsources.html#eventsources-api-gateway-request>`_.

With Minik, you get a clear familiar interface that hides the complexity of dealing
with the raw representation of the event and context objects. We take care of parsing
the API gateway object for you, so that you can focus on writing your business logic.
Using the above object and endpoint as an example, our lambda function would instead be:

.. code:: python

    from minik.core import Minik
    app = Minik()

    @app.route("/test/{action}")
    def hello(action):
        name = app.request.query_params.get('name')

        # With the values defined in the object above this will return.
        # {'hello': 'me'}
        return {action: name}


Just like with any other lambda function you are responsible for provisioning the
API Gateway and for associating the lambda function with the gateway endpoint. Minik
is just the framework that allows you to write your api in a straight-forward fashion.


Minik with ALB
**************
When working with a lambda function as the handler of a request from an Application
Load Balancer (ALB), a simple hello world application looks like:

.. code:: python

    def lambda_handler(event, context):
        response = {
            "statusCode": 200,
            "statusDescription": "200 OK",
            "isBase64Encoded": False,
            "headers": {
            "Content-Type": "text/html; charset=utf-8"
            }
        }

        response['body'] = """<html>
            <head>
            <title>Hello World!</title>
            <style>
            html, body {
            margin: 0; padding: 0;
            font-family: arial; font-weight: 700; font-size: 3em;
            text-align: center;
            }
            </style>
            </head>
            <body>
            <p>Hello World!</p>
            </body>
            </html>"""
        return response

In this scenario, the event your lambda function receives from the ALB looks like this:

.. code:: json

    {
        "requestContext": {
            "elb": {
                "targetGroupArn": "arn:aws:elasticloadbalancing:us-east-1:XXXXXXXXXXX:targetgroup/sample/6d0ecf831eec9f09"
            }
        },
        "httpMethod": "GET",
        "path": "/",
        "queryStringParameters": {},
        "headers": {
            "accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
            "accept-encoding": "gzip",
            "accept-language": "en-US,en;q=0.5",
            "connection": "keep-alive",
            "cookie": "name=value",
            "host": "lambda-YYYYYYYY.elb.amazonaws.com",
            "upgrade-insecure-requests": "1",
            "user-agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.11; rv:60.0) Gecko/20100101 Firefox/60.0",
            "x-amzn-trace-id": "Root=1-5bdb40ca-556d8b0c50dc66f0511bf520",
            "x-forwarded-for": "192.0.2.1",
            "x-forwarded-port": "80",
            "x-forwarded-proto": "http"
        },
        "body": "",
        "isBase64Encoded": false
    }

Without Minik, every single API endpoint will need to parse the raw object
as a way to get the data values you care about.

With Minik, you get a clear familiar interface that hides the complexity of dealing
with the raw representation of the event and context objects. We take care of parsing
the ALB object for you, so that you can focus on writing your business logic.
Using the above object and endpoint as an example, our lambda function would instead be:

.. code:: python

    from minik.core import Minik
    app = Minik()

    @app.route("/test/{action}")
    def hello(action):
        name = app.request.query_params.get('name')

        # With the values defined in the object above this will return.
        # {'hello': 'me'}
        return {action: name}


Just like with any other lambda function you are responsible configuring the lambda
function as the `target of the ALB`_. Minik is just the framework that facilitates the
definition of the web application.

Notice that the code to handle a request from the API Gateway is identical to the
code used in the ALB example. This shows that minik is service agnostic, an web
application that was associated to an API Gateway definition, can seamlessly be
used to handle requests from an ALB without changing the code.

.. _`target of the ALB`: https://aws.amazon.com/blogs/networking-and-content-delivery/lambda-functions-as-targets-for-application-load-balancers/

.. _SAM: https://github.com/awslabs/serverless-application-model
.. _Chalice: https://github.com/aws/chalice
.. _Serverless: https://serverless.com/
.. _Juniper: https://github.com/eabglobal/juniper

.. |circle| image:: https://circleci.com/gh/eabglobal/minik/tree/master.svg?style=shield
    :target: https://circleci.com/gh/eabglobal/minik/tree/master

.. |pypi version| image:: https://img.shields.io/pypi/v/minik.svg
    :target: https://pypi.org/project/minik/

.. |apache license| image:: https://img.shields.io/github/license/eabglobal/minik.svg
    :target: https://github.com/eabglobal/minik/blob/master/LICENSE

Contents:
=========

.. toctree::
    :maxdepth: 2
    :numbered:
    :glob:

    quickstart.rst
    features.rst
