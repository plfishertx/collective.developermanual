=================
Cache decorators
=================

.. admonition:: Description

    How to use the Python decorator pattern to cache the result values of
    your computationally expensive method calls.

.. contents:: :local:

Introduction
============

Cache decorators are convenient methods caching of function return values.

Use them like this::

    @cache_this_function
    def my_slow_function():
        # This is run only once and all subsequent calls get value from the cache
        return  

.. warning::

    Cache decorators do not work with methods or functions that use
    generators (``yield``).
    The cache will end up storing an empty value.

The `plone.memoize <http://pypi.python.org/pypi/plone.memoize>`_ package
offers helpful function decorators to cache return values.

See also :doc:`using memcached backend for memoizers </performance/ramcache>`. 

Cache result for process lifecycle
==================================

Example::

    from plone.memoize import forever

    @forever.memoize
    def getFields(area, subject):
        """ Get all fields inside area / subject.

        Results is cached for process lifetime.

        @return: List of internal fields
        """
        schema = getSchema(area)
        return [field for field in schema if field["subject"] == subject]


Timeout caches
==============

The @ram.cache decorator takes a function argument and calls it to get a value. 
So long as that value is unchanged, the cached result of the decorated function is returned.
This makes it easy to set a timeout cache::

    from plone.memoize import ram
    from time import time

    @ram.cache(lambda *args: time() // (60 * 60))
    def cached_query(self):
        # very expensive operation, 
        # will not be called more than once an hour

time.time() returns the time in seconds as a floating point number. "//" is Python's integer division.
So, the result of ``time() // (60 * 60)`` only changes once an hour. 
``args`` passed are ignored.


Caching per request
===================

This pattern shows how to avoid recalculating the same value repeatedly
during the lifecycle of an HTTP request.

Caching on BrowserViews
------------------------

This is useful if the same view/utility is going to be called many times
from different places during the same HTTP request.

The `plone.memoize.view <https://github.com/plone/plone.memoize/tree/master/plone/memoize/view.txt>`_
package provides necessary decorators for ``BrowserView``-based classes.

.. code-block:: python

    from plone.memoize.view import memoize, memoize_contextless

    class MyView(BrowserView):

        @memoize
        def getValue():
            """ This value is recalculated for every new BrowserView context
                per request.
            """
            return "something"

        @memoize_contextless
        def getValueNoContext():
            """ This value is recalculated for all context objects once per
                request.
            """
            return "something"

Caching on Archetypes accessors
---------------------------------

If you have a custom 
:doc:`Archetypes accessor method </content/archetypes/fields>`,
you can avoid recalculating it during the request processing.

Example::

    def getParsedORADataCached(self):
        """ Same as above, but does not run through JSON reader every time.
        """

        # Manually store the result on HTTP request object annotations 

        # Use informative string + Archetypes unique identified as the key
        key = "parsed-ora-data-" + self.UID()

        cache = IAnnotations(self.REQUEST)
        data = cache.get(key, None)
        if data is not None:
            data = self.getParsedORAData()
            cache[key] = data 

        return data

Caching using global HTTP request
----------------------------------

This example uses the 
`five.globalrequest package <http://pypi.python.org/pypi/five.globalrequest>`_ 
for caching. Values are stored on the thread-local ``HTTPRequest`` object
which lasts for the transaction lifecycle::

    from zope.globalrequest import getRequest
    from zope.annotation.interfaces import IAnnotations

        def _getProductList(self, type, language):
            """ Private implementation, builds list of products.
            """

            logger.info("Getting product list %s %s" % (type, language))
            ...
            return result


        def getProductListCached(self, type, language):
            """ Public cached method, delegates to _getProductList.
            """

            request = getRequest()

            key = "cache-%s-%s" % (type, language)

            cache = IAnnotations(request)
            data = cache.get(key, None)
            if not data:
                data = self._getProductList(type, language)
                cache[key] = data

            return data


Other resources
===============

* `plone.memoize source code <https://github.com/plone/plone.memoize/tree/master/plone/memoize/>`_

* `zope.app.cache source code <http://svn.zope.org/zope.app.cache/trunk/src/zope/app/cache/>`_


