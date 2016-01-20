.. _tutorial-write-plugin:

How to write a Kinto plugin
###########################

*Kinto* plugins allow to add extra-features to *Kinto*. Most notably:

* Respond to internal events (e.g. notify third-party)
* Add endpoints for custom URLs (e.g. new hook URL)
* Add custom endpoint renderers (e.g. XML instead of JSON)

As :ref:`specified in the settings section <configuration-plugins>`, the
are Python module loaded on startup.

In this tutorial, we will build an *ElasticSearch* plugin that will:

* Initialize an indexer on startup;
* Add a ``/{collection}/search`` end-point;
* Index records when created/updated/deleted.


Include me
----------

First, create a Python package and install it locally. For example:

::

    $ pip install cookiecutter
    $ cookiecutter gh:kragniz/cookiecutter-pypackage-minimal

    $ cd kinto-elasticsearch
    $ python setup.py develop


In order to be included, a package must define an ``includeme(config)`` function.

For example, in :file:`kinto_elasticsearch/init.py`:

.. code-block:: python

    def includeme(config):
        print("I am ElasticSearch plugin")


Add it the :file:`config.ini` file:

.. code-block:: ini

    kinto.includes = kinto_elasticsearch

Our message should not appear on ``kinto start``.


Simple indexer
--------------

Let's define a simple indexer class in :file:`kinto_elasticsearch/indexer.py`.
It can search and index records.

.. code-block:: python

    import elasticsearch

    class Indexer(object):
        def __init__(self, hosts):
            self.client = elasticsearch.Elasticsearch(hosts)

        def search(self, bucket_id, collection_id, query, **kwargs):
            indexname = '%s-%s' % (bucket_id, collection_id)
            try:
                return self.client.search(index=indexname,
                                          doc_type=indexname,
                                          body=query,
                                          **kwargs)
            except exception.ElasticSearchException:
                return {}

        def index_record(self, bucket_id, collection_id, record, id_field):
            indexname = '%s-%s' % (bucket_id, collection_id)
            record_id = record[id_field]
            index = self.client.index(index=indexname,
                                      doc_type=indexname,
                                      id=record_id,
                                      body=record,
                                      refresh=True)
            return index

        def unindex_record(self, bucket_id, collection_id, record, id_field):
            indexname = '%s-%s' % (bucket_id, collection_id)
            record_id = record[id_field]
            result = self.client.delete(index=indexname,
                                        doc_type=indexname,
                                        id=record_id,
                                        refresh=True)
            return result


And a simple method to load from configuration:

.. code-block:: python

    from pyramid.settings import aslist

    def load_from_config(config):
        settings = config.get_settings()
        hosts = aslist(settings.get('elasticsearch.hosts', 'localhost:9200'))
        indexer = Indexer(hosts=hosts)
        return indexer


Initialize on startup
---------------------

.. code-block:: python
    :emphasize-lines: 4

    from . import indexer

    def includeme(config):
        # Register a global indexer object
        config.registry.indexer = indexer.load_from_config(config)


Add a search view
-----------------

Add an end-point definition in :file:`kinto_elasticsearch/views.py`:

.. code-block:: python

    from cliquet import Service

    search = Service(name="search",
                     path='/buckets/{bucket_id}/collections/{collection_id}/search',
                     description="Search")

    @search.post()
    def get_search(request):
        bucket_id = request.matchdict['bucket_id']
        collection_id = request.matchdict['collection_id']

        query = request.body

        # Access indexer from views using registry.
        indexer = request.registry.indexer
        results = indexer.search(bucket_id, collection_id, query)

        return results

Enable the view:

.. code-block:: python
    :emphasize-lines: 6,7

    from . import indexer

    def includeme(config):
        # Register a global indexer object
        config.registry.indexer = indexer.load_from_config(config)

        # Activate end-points.
        config.scan('kinto_elasticsearch.views')

This new URL should now be able to return results from ElasticSearch.

::

     $ http POST "http://localhost:8888/v1/buckets/default/collections/articles/search


Index records on change
-----------------------

When a record changes, we update its indexed version:


.. code-block:: python

    def on_resource_changed(event):
        indexer = event.request.registry.indexer

        resource_name = event.payload['resource_name']

        if resource_name != "record":
            return

        bucket_id = event.payload['bucket_id']
        collection_id = event.payload['collection_id']

        action = event.payload['action']
        for change in events.impacted_records:
            if action == 'delete':
                index.unindex_record(bucket_id,
                                     collection_id,
                                     record=change['old'],
                                     id='id')
            else:
                index.index_record(bucket_id,
                                   collection_id,
                                   record=change['old'],
                                   id='id')

And then we bind:

.. code-block:: python
    :emphasize-lines: 1,13,14

    from cliquet.events import ResourceChanged

    from . import indexer

    def includeme(config):
        # Register a global indexer object
        config.registry.indexer = indexer.load_from_config(config)

        # Activate end-points.
        config.scan('kinto_elasticsearch.views')

        # Plug the callback with resource events.
        config.add_subscriber(on_resource_changed, ResourceChanged)



Test it altogether
------------------

::

    $ echo '{"data": {"note": "kinto"}}' | http --auth alice: --verbose --form POST http://localhost:8888/v1/buckets/default/collections/assets/records



::

    $ http --auth alice: --verbose --form POST http://localhost:8888/v1/buckets/default/collections/assets/search

