flask-appcache
==============

I've spent enough time tracking down appcache issues now.. Let's do this the
right way.

[![Build Status](https://travis-ci.org/shuhaowu/flask-appcache.png)](https://travis-ci.org/shuhaowu/flask-appcache)


Install
-------

    $ pip install Flask-Appcache

Quick Example
-------------

This is really easy to use. If not, file an issue! Here's an overview:

Suppose if we start with a standard flask app with the following directory
structures.

    server.py
    templates/
      - manifest.appcache
    static
      - app.js
      - app.css
      - img.png

Here's server.py:

    from flask import Flask
    from flask.ext.appcache import Appcache

    app = Flask(__name__)

    # initializes the appcache
    # you can also use the standard init_app function instead of initializing
    # class with the app
    appcache = Appcache(app)

    # adds some urls to cache.
    appcache.add_urls("/", "/offline")
    appcache.add_folder("/absolute/path/is/recommended/static", base="/static")

    @app.route("/")
    def main():
      return "Yay I'm cached!"

    @app.route("/offline")
    def offline():
      return "awe i'm offline"

Here's an example manifest.appcache file (actually this is probably good for
most purposes):

    CACHE MANIFEST
    # version {{ hash }}
    # It is recommended that you don't have the line below if you run multiple
    # processes as they will each have a different time.
    # updated {{ updated }}

    CACHE:
    {% for url in urls %}{{ url }}
    {% endfor %}

    FALLBACK:
    / /offline

    NETWORK:
    *

Flask-Appcache registers a route for you, /manifest.appcache, to render and
serve the template as specified above. Here's an example rendered version of
this file:

    CACHE MANIFEST
    # version ab1a2034bc23a734015d18c06c4d29f2264d7a8e
    # It is recommended that you don't have the line below if you run multiple
    # processes as they will each have a different time.
    # updated 2013-08-28T15:16:34.663515

    CACHE:
    /
    /offline
    /static/app.js
    /static/app.css
    /static/img.png

    FALLBACK:
    / /offline

    NETWORK:
    *

The hash will change if you change any of the files (including the rendered
views like / and /offline). updated will be as well allowing the app to be
updated on the client side.

More advanced features
----------------------

You can exclude URLs from being tracked even if you tell the system to add it
(an example use case would be blacklisting some urls from a folder when you
add a folder). Exclusion URLs are using a prefix scheme. So if there are any
urls that has the prefix of an excluded url, it will be excluded. To use this
feature:

    # This will ignore /static/js/ignored/1.js, /static/js/ignored/2.js and
    # so on. Pretty much anything that starts with /static/js/ignored
    appcache.add_excluded_urls("/static/js/ignored")

You can add folders but specify a different base url. An example usecase would
be if you changed the static folder:

    # This will remap any files under static to have a /media prefix.
    # static/js/1.js => /media/js/1.js
    # static/css/app.css => /media/css/app.css
    appcache.add_folder("static", base="/media")

You can change the url where manifest.appcache is served by specifying a config
before you initialize the appcache.

    # defaults to /manifest.appcache
    app.config["APPCACHE_URL"] = "/my_manifest.appcache"
    appcache = Appcache(app)
    ....

You can also change the template name:

    # make sure that you have the file
    app.config["APPCACHE_TEMPLATE"] = "my_manifest.appcache"
    appcache = Appcache(app)

Some notes here and there
-------------------------

Flask-Appcache registers an `after_request` handler that changes all cache
control to `must-revalidate`. This might not be the ideal behaviour so it could
be changed that that head is only emitted for the manifest.appcache route and
any url routes tracked by appcache.

In order to compute the hash of the current site, Flask-Appcache makes a fake
request to your server to retrieve the content of an appcache'd URL. This makes
it play well with any view. Note, this view should not change often as it will
screw up appcache updates. This happens everytime manifest.appcache is loaded
in debug mode.

Deployment considerations
-------------------------

As previously stated, everytime manifest.appcache is requested, the framework
goes and hashes every copy of the file that needs to be appcached so it can
hash it and check for changes. This would be slow. Luckily, there is a
`finalize` method that precomputes the hashes.

Here's an example deployment configuration to be put in place in server.py:

    if app.debug:
      # exclude urls associated with deployment
      appcache.add_excluded_urls("/static/js/app.min.js",
                                 "/static/css/app.min.css")
    else:
      # excludes all the urls associated with development
      appcache.add_excluded_urls("/static/js/develop",
                                 "/static/css/develop")

    # adds all the static files
    appcache.add_folder("static", base="/static")
    if not app.debug:
      # precomputes the hashes and stays there
      appcache.finalize()

API
---

Help on module flask_appcache:

    NAME
        flask_appcache

    CLASSES
        __builtin__.object
            Appcache

        class Appcache(__builtin__.object)
         |  Methods defined here:
         |
         |  __init__(self, app=None)
         |      Initializes a new instance of Appcache
         |
         |  add_excluded_urls(self, *urls)
         |      Adds urls to exclude from appcache.
         |
         |      Args:
         |        urls: the urls to cache
         |
         |  add_folder(self, folder, base='/static')
         |      Adds a whole folder to appcache.
         |
         |      Args:
         |        folder: the folder's content to cache.
         |        base: The base url.
         |
         |        As an example, if you have media/ being your static dir and that is
         |        mapped to the server url of /static, you would do
         |        add_folder('media', base='/static')
         |
         |  add_urls(self, *urls)
         |      Adds individual urls on appcache.
         |
         |      Args:
         |        urls: The urls to be appcached.
         |
         |  finalize(self)
         |      Finalizes the appcache by precomputing the hash.
         |
         |      This will cause the hash() function to never compute the hash again.
         |
         |  hash(self)
         |      Computes the hash and the last updated time for the current appcache
         |
         |      Returns:
         |        hash, time updated in isoformat
         |
         |  init_app(self, app)
         |      Initializes the app.
         |
         |      Warning: set all the config variables before this so the correct route will
         |      be registered.
         |
         |  ----------------------------------------------------------------------
         |  Data descriptors defined here:
         |
         |  __dict__
         |      dictionary for instance variables (if defined)
         |
         |  __weakref__
         |      list of weak references to the object (if defined)


