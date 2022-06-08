JavaScript, ``fetch``, and JSON
===============================

You may want to make your HTML page dynamic, by changing data without
reloading the entire page. Instead of submitting an HTML ``<form>`` and
performing a redirect to re-render the template, you can add
`JavaScript`_ that calls `|fetch|`_ and replaces content on the page.

`|fetch|`_ is the modern, built-in JavaScript solution to making
requests from a page. You may have heard of other "AJAX" methods and
libraries, such as `|XHR|`_ or `jQuery`_. These are no longer needed in
modern browsers, although you may choose to use them or another library
depending on your application's requirements. These docs will only focus
on built-in JavaScript features.

.. _JavaScript: https://developer.mozilla.org/en-US/docs/Web/JavaScript
.. |fetch| replace:: ``fetch()``
.. _fetch: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
.. |XHR| replace:: ``XMLHttpRequest()``
.. _XHR: https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest
.. _jQuery: https://jquery.com/


Rendering Templates
-------------------


Generating URLs
---------------

The simplest way to generate URLs is to continue to use
:func:`~flask.url_for` when rendering the template. For example:

.. code-block:: javascript

    const user_url = {{ url_for("user", id=current_user.id)|tojson }}
    fetch(user_url).then(...)
    // or inline
    fetch({{ url_for("user", id=current_user.id)|tojson }}).then(...)

However, you might need to generate a URL based on information you only
know in JavaScript. As discussed above, JavaScript runs in the user's
browser, not as part of the template rendering, so you can't use
``url_for`` at that point.

In this case, you need to know the "root URL" under which your
application is served. In simple setups, this is ``/``, but it might
also be something else, like ``https://example.com/myapp``.

A simple way to tell your JavaScript code about this root is to set it
as a global variable when rendering the template. Then you can use it
when generating URLs from JavaScript.

.. code-block:: javascript

    const SCRIPT_ROOT = {{ request.script_root|tojson }}
    let user_id = ...  // do something to get a user id from the page
    fetch(`${SCRIPT_ROOT}/user/${user_id}`).then(...)

Do you know where your application is?  If you are developing the answer
is quite simple: it's on localhost port something and directly on the root
of that server.  But what if you later decide to move your application to
a different location?  For example to ``http://example.com/myapp``?  On
the server side this never was a problem because we were using the handy
:func:`~flask.url_for` function that could answer that question for
us, but if we are using jQuery we should not hardcode the path to
the application but make that dynamic, so how can we do that?

A simple method would be to add a script tag to our page that sets a
global variable to the prefix to the root of the application.  Something
like this:

.. sourcecode:: html

    <script>
      const SCRIPT_ROOT = {{ request.script_root|tojson }};
    </script>


JSON View Functions
-------------------

Now let's create a server side function that accepts two URL arguments of
numbers which should be added together and then sent back to the
application in a JSON object.  This is a really ridiculous example and is
something you usually would do on the client side alone, but a simple
example that shows how you would use jQuery and Flask nonetheless::

    from flask import Flask, jsonify, render_template, request
    app = Flask(__name__)

    @app.route('/_add_numbers')
    def add_numbers():
        a = request.args.get('a', 0, type=int)
        b = request.args.get('b', 0, type=int)
        return jsonify(result=a + b)

    @app.route('/')
    def index():
        return render_template('index.html')

As you can see I also added an `index` method here that renders a
template.  This template will load jQuery as above and have a little form where
we can add two numbers and a link to trigger the function on the server
side.

Note that we are using the :meth:`~werkzeug.datastructures.MultiDict.get` method here
which will never fail.  If the key is missing a default value (here ``0``)
is returned.  Furthermore it can convert values to a specific type (like
in our case `int`).  This is especially handy for code that is
triggered by a script (APIs, JavaScript etc.) because you don't need
special error reporting in that case.

The HTML
--------

Your index.html template either has to extend a :file:`layout.html` template with
jQuery loaded and the `$SCRIPT_ROOT` variable set, or do that on the top.
Here's the HTML code needed for our little application (:file:`index.html`).
Notice that we also drop the script directly into the HTML here.  It is
usually a better idea to have that in a separate script file:

.. sourcecode:: html

    <script>
      $(function() {
        $('a#calculate').bind('click', function() {
          $.getJSON($SCRIPT_ROOT + '/_add_numbers', {
            a: $('input[name="a"]').val(),
            b: $('input[name="b"]').val()
          }, function(data) {
            $("#result").text(data.result);
          });
          return false;
        });
      });
    </script>
    <h1>jQuery Example</h1>
    <p><input type=text size=5 name=a> +
       <input type=text size=5 name=b> =
       <span id=result>?</span>
    <p><a href=# id=calculate>calculate server side</a>

I won't go into detail here about how jQuery works, just a very quick
explanation of the little bit of code above:

1. ``$(function() { ... })`` specifies code that should run once the
   browser is done loading the basic parts of the page.
2. ``$('selector')`` selects an element and lets you operate on it.
3. ``element.bind('event', func)`` specifies a function that should run
   when the user clicked on the element.  If that function returns
   `false`, the default behavior will not kick in (in this case, navigate
   to the `#` URL).
4. ``$.getJSON(url, data, func)`` sends a ``GET`` request to `url` and will
   send the contents of the `data` object as query parameters.  Once the
   data arrived, it will call the given function with the return value as
   argument.  Note that we can use the `$SCRIPT_ROOT` variable here that
   we set earlier.

Check out the :gh:`example source <examples/javascript>` for a full
application demonstrating the code on this page, as well as the same
thing using ``XMLHttpRequest`` and ``fetch``.
