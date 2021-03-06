Feature Tour
============


Class-Based Views
-----------------

Class-based views (and setting some headers and stuff)::

    @api.route("/{greeting}")
    class GreetingResource:
        def on_request(req, resp, *, greeting):   # or on_get...
            resp.text = f"{greeting}, world!"
            resp.headers.update({'X-Life': '42'})
            resp.status_code = api.status_codes.HTTP_416


Background Tasks
----------------

Here, you can spawn off a background thread to run any function, out-of-request::

    @api.route("/")
    def hello(req, resp):

        @api.background.task
        def sleep(s=10):
            time.sleep(s)
            print("slept!")

        sleep()
        resp.content = "processing"


GraphQL
-------

Serve a GraphQL API::

    import graphene

    class Query(graphene.ObjectType):
        hello = graphene.String(name=graphene.String(default_value="stranger"))

        def resolve_hello(self, info, name):
            return f"Hello {name}"

    api.add_route("/graph", graphene.Schema(query=Query))

Visiting the endpoint will render a *GraphiQL* instance, in the browser.


OpenAPI Schema Support
----------------------

Responder comes with built-in support for OpenAPI / marshmallow::

    import responder
    from marshmallow import Schema, fields

    api = responder.API(title="Web Service", version="1.0", openapi="3.0")


    @api.schema("Pet")
    class PetSchema(Schema):
        name = fields.Str()


    @api.route("/")
    def route(req, resp):
        """A cute furry animal endpoint.
        ---
        get:
            description: Get a random pet
            responses:
                200:
                    description: A pet to be returned
                    schema:
                        $ref = "#/components/schemas/Pet"
        """
        resp.media = PetSchema().dump({"name": "little orange"})


::

    >>> r = api.session().get("http://;/schema.yml")

    >>> print(r.text)
    components:
    parameters: {}
    schemas:
        Pet:
        properties:
            name: {type: string}
        type: object
    info: {title: Web Service, version: 1.0}
    openapi: '3.0'
    paths:
      /:
        get:
          description: Get a random pet
          responses:
            200: {description: A pet to be returned, schema: $ref = "#/components/schemas/Pet"}
    tags: []


Interactive Documentation
-------------------------

Responder can automatically supply API Documentation for you. Using the example above::

    api = responder.API(title="Web Service", version="1.0", openapi="3.0", docs_route="/docs")

This will make ``/docs`` render interactive documentation for your API.

Mount a WSGI App (e.g. Flask)
-----------------------------

Responder gives you the ability to mount another ASGI / WSGI app at a subroute::

    import responder
    from flask import Flask

    api = responder.API()
    flask = Flask(__name__)

    @flask.route('/')
    def hello():
        return 'hello'

    api.mount('/flask', flask)

That's it!

Single-Page Web Apps
--------------------

If you have a single-page webapp, you can tell Responder to serve up your ``static/index.html`` at a route, like so::

    api.add_route("/", static=True)

This will make ``index.html`` the default response to all undefined routes.

Reading / Writing Cookies
-------------------------

Responder makes it very easy to interact with cookies from a Request, or add some to a Response::

    >>> resp.cookies["hello"] = "world"

    >>> req.cookies
    {"hello": "world"}


Using Cookie-Based Sessions
---------------------------

Responder has built-in support for cookie-based sessions. To enable cookie-based sessions, simply add something to the ``resp.session`` dictionary::

    >>> resp.session['username'] = 'kennethreitz'

A cookie called ``Responder-Session`` will be set, which contains all the data in ``resp.session``. It is signed, for verification purposes.

You can easily read a Request's session data, that can be trusted to have originated from the API::

    >>> req.session
    {'username': 'kennethreitz'}

**Note**: if you are using this in production, you should pass the ``secret_key`` argument to ``API(...)``::

    api = responder.API(secret_key=os.environ['SECRET_KEY'])

Using Requests Test Client
--------------------------

Responder comes with a first-class, well supported test client for your ASGI web services: **Requests**.

Here's an example of a test (written with pytest)::

    import myapi

    @pytest.fixture
    def api():
        return myapi.api

    def test_response(api):
        hello = "hello, world!"

        @api.route('/some-url')
        def some_view(req, resp):
            resp.text = hello

        r = api.requests.get(url=api.url_for(some_view))
        assert r.text == hello

HSTS (Redirect to HTTPS)
------------------------

Want HSTS (to redirect all traffic to HTTPS)?

::

    api = responder.API(enable_hsts=True)


Boom.
