======
ebdata
======

Code to help write scripts that import/crawl/parse data from the web
into ebpub, as well as extract addresses from (English) text.

Scraper scripts will probably be built on either ebdata.retrieval_ or
:py:mod:`ebdata.blobs`, depending on the type of content being scraped.

ebdata.nlp
===========

Address extraction from raw text.  For details see
:py:mod:`ebdata.nlp.addresses` particularly the 
:py:func:`parse_addresses <ebdata.nlp.addresses.parse_addresses>` function.

ebdata.retrieval
================

A framework for writing scrapers for structured
data. Some examples can be found in
ebdata.scrapers_.  There are more (unmaintained) examples of how to use this
framework in different situations in the ``everyblock`` package (see :ref:`other_packages`).

(For scraping data from unstructured sites, eg. sites that lack feeds
or machine-consumable API, you might consider building on the
:py:mod:`ebdata.blobs` package.)

The most commonly used scraper base class is the
``NewsItemListDetailScraper``. It handles scraping list/detail types
of sites, and creating or updating NewsItem objects.  "List" could be
an RSS or Atom feed, or an HTML index, which links to "detail" pages;
these can be any format, such as HTML, XML, or JSON.  (In some cases,
the feed provides all the necessary information, and there's no need
to retrieve any detail pages.)

Generally, to run a scraper, you need to instantiate it, and then call its
``update()`` method. Sometimes the scraper will take arguments, but it varies on a
case-by-case basis; see the scrapers in ebdata.scrapers_ for
examples. You can also run a scraper by calling its ``display_data()`` method. This
will run the scraper, but won't actually save any of the scraped data. It's
very useful for debugging, or when writing a scraper for the first time.

All of the methods and parameters you'll need to use are documented in
docstrings of ``ebdata.retrieval.scrapers.list_detail.ListDetailScraper`` and in
``ebdata.retrieval.scrapers.newsitem_list_detail.NewsItemListDetailScraper``.
``ListDetailScraper`` is a base class that handles
scraping, but doesn't actually have any methods for saving data.

(TODO: Document the base class methods and their interactions here)

.. _ebdata-scrapers:

ebdata.scrapers
===============

A collection of ready-to-run scraper scripts, with JSON fixture files
for loading the schemas needed by each scraper.

(If you want to write your own scrapers for other data sources, see
:doc:`../main/scraper_tutorial`.)

These generally leverage the tools in ebdata.retrieval.

All of them can be run as command-line scripts. Use the ``-h`` option to
see what options, if any, each script takes.

See :doc:`../main/running_scrapers` for more about running scrapers in
general, and :doc:`../main/scraper_tutorial` for more about writing
your own scrapers.

.. _ebdata-georss:

RSS Feeds: scrapers.general.georss
---------------------------------------

Loads any RSS or Atom feed from a URL.

Usage::

  $VIRTUAL_ENV/bin/python ebdata/scrapers/general/georss/retrieval.py [options] <feed url>
  
  Options:
   -h, --help       Show this help message and exit
   --schema=SCHEMA  Slug of the news item type to create when scraping.
   -v, --verbose    Verbose output.
   -q, --quiet      No output.

This will fetch data from the given URL and store it in NewsItems of
the specified Schema.

This scraper uses only the
:ref:`generic core NewsItem fields <newsitem_core_fields>`
and does not populate any :ref:`extra custom attributes <newsitem_attributes>`.
This is fine for most RSS feeds, since they typically don't contain
any information beyond what would go directly in a plain vanilla NewsItem.

In typical usage, where each feed URL represents a different kind of
news, you would create one :ref:`Schema <newsitem-schemas>` for each
feed URL, but no SchemaFields.

More Complex Feeds
~~~~~~~~~~~~~~~~~~~

If you *do* have a feed with extra data elements, or need to split up
or otherwise process information from some of the generic RSS or Atom
elements, you would need to write your own scraper, possibly using this
code as a starting point.

An example is discussed at :ref:`scraping_listdetail`.

Location Parsing
~~~~~~~~~~~~~~~~

The RSS scraper tries to extract a point location and a location name
(eg. address) from the feed, according to the following strategy:

* First it looks for a latitude/longitude point expressed in several
  standard formats:
  `GeoRSS (simple or GML) <http://www.georss.org/Main_Page>`_,
  `RDF geo <http://www.w3.org/2003/01/geo/>`_,
  `xCal <http://tools.ietf.org/html/rfc6321>`_.
* Next it tries some common but non-standardized tags such as
  ``<latitude>`` and ``<longitude>``.
* If no point is found, it looks for a location name in
  standard GeoRSS or xCal elements; if found, geocode that to derive a point.
* If no location name is found, it tries to extract and geocode any
  addresses or location names from any tags that contain text.
  (Uses :ref:`ebdata-nlp`)
* If a point was found, but a location name was not,
  it tries to reverse-geocode the point to derive an address.
* If all of the above fail, it does not create a NewsItem.

The scraper script is ``PATH/TO/ebdata/scrapers/general/georss/retrieval.py``
and a generic "local news" schema can be loaded by doing
``django-admin.py loaddata PATH/TO/ebdata/scrapers/general/georss/local_news_schema.json``.

If you want to use another schema, you can give the ``--schema``
command-line option.

.. _spreadsheet_scraper:

Spreadsheets: scrapers.general.spreadsheet
---------------------------------------------------

This scraper can handle many CSV spreadsheets.


Usage::

 retrieval.py [options] <spreadsheet> [<mapping spreadsheet>]
 
 Spreadsheet arguments can be local files or URLs.
 
 See http://openblockproject.org/docs/packages/ebdata.html#spreadsheets-scrapers-general-spreadsheet for more.
 
 Options:
  -h, --help            show this help message and exit
  --schema=SCHEMA       slug of news item type to create when scraping
  --unique-fields=UNIQUE_FIELDS
                        Which NewsItem fields identify a unique record in this
                        data source. Comma-separated, eg. --unique-
                        fields='url,location_name,title
  -v, --verbose         Verbose output.
  -q, --quiet           No output.

.. admonition:: Alternative: Importing spreadsheets via the admin UI

  If you point your browser at /admin/db/newsitem/ you can manually
  upload spreadsheets using a form. It works much the same way as this
  scraper.  Read more at :ref:`newsitem_upload`.

  The admin UI might be easier if you get spreadsheets
  rarely, or if you just want to manually create a large number of
  NewsItems at once.


The scraper script is ``PATH/TO/ebdata/scrapers/general/spreadsheet/retrieval.py``
and a generic "local news" schema can be loaded by doing
``django-admin.py loaddata PATH/TO/ebdata/scrapers/general/georss/local_news_schema.json``.  

Any rows that don't yield valid NewsItems will be skipped.

The ``--schema`` command-line option defaults to "local-news".

The script takes one or two positional arguments.
The first is the spreadsheet containing NewsItem data, which may be a
local file or a URL.  The second is an optional "mapping" spreadsheet explaining
how to interpret the data in the first spreadsheet. Details follow.

.. include:: ../spreadsheet.rst

Example:

.. code-block:: bash

  python ebdata/scrapers/general/spreadsheet/retrieval.py \
    --unique-fields=title,item_date \
    http://example.com/spreadsheet.csv


Flickr: scrapers.general.flickr
---------------------------------------

Usage::

 $VIRTUAL_ENV/bin/python ebdata/scrapers/general/flickr/flickr_retrieval.py [options]

  Options:
   -h, --help            show this help message and exit
   -d DAYS, --days=DAYS  How many days (prior to stop date) to search. Default
                         is 30 days.
   -e END_DATE, --end-date=END_DATE
                         Stop date for photo search, format YYYY/MM/DD. Default
                         is now.
   --schema=SCHEMA       Slug of schema to use. Default is 'photos'.
   -v, --verbose         Verbose output.
   -q, --quiet           No output.


Loads Flickr photos that are geotagged at a location within your
configured :ref:`metro extent <metro_extent>`.

You must set both ``settings.FLICKR_API_KEY`` and ``settings.FLICKR_API_SECRET``.

You must also install a library that it depends on::

  $ $VIRTUAL_ENV/bin/pip install flickrapi

(Note that if :doc:`obdemo` is installed, you should already have this
library.)

The scraper script is ``PATH/TO/ebdata/scrapers/general/flickr/flickr_retrieval.py``
and the schema can be loaded by doing
``django-admin.py loaddata PATH/TO/ebdata/scrapers/general/flickr/photos_schema.json``.

Meetup: scrapers.general.meetup
---------------------------------------

Retrieves upcoming Meetups from `meetup.com <http://meetup.com>`_.  USA-only.
This assumes you have loaded some :ref:`zipcodes`,
as it will attempt to load meetups for all your zip codes.

Usage::

 $VIRTUAL_ENV/bin/python ebdata/scrapers/general/meetup/meetup_retrieval.py [options]

 Options:
  -h, --help            show this help message and exit
  --schema=SCHEMA       Slug of schema to use. Default is 'meetups'.
  -p START_PAGE, --start-page=START_PAGE
                        Page of results to start from. Default is zero.
  -n, --no-wait-for-rate-limit
                        If we hit rate limit, exit instead of waiting until it
                        resets (typically 1 hour). Default is to wait.
  -v, --verbose         Verbose output.
  -q, --quiet           No output.



You will need to get an API key, and set it as ``settings.MEETUP_API_KEY``.


The scraper script is ``PATH/TO/ebdata/scrapers/general/meetup/meetup_retrieval.py``
and the schema can be loaded by doing
``django-admin.py loaddata PATH/TO/ebdata/scrapers/general/meetup/meetup_schema.json``.

This scraper may take hours to run, since Meetup's API has a rate
limit of 200 requests per hour (returning up to 200 meetups each), and
a large city may have thousands of meetups every day, and we're trying
to load all scheduled meetups for the next few months. The default
behavior is to run until the API's rate limit is hit, then wait till
the limit is lifted (typically 1 hour), and repeat until all pages for
all zip codes have been loaded.  If you'd rather do smaller batches,
try the ``--help`` option to see what options you have.


Open311 / GeoReport: scrapers.general.open311
------------------------------------------------------

A scraper for the
`Open311 / GeoReport API <http://wiki.open311.org/GeoReport_v2#GET_Service_Requests>`_
that is being adopted by a
`growing number of cities <http://wiki.open311.org/GeoReport_v2/Servers>`_
including many served by `SeeClickFix <http://seeclickfix.com>`.

Usage::

 $VIRTUAL_ENV/bin/python ebdata/scrapers/general/open311/georeportv2.py [options] <api url>
 
 Options:
  -h, --help            show this help message and exit
  -k API_KEY, --api-key=API_KEY
                        GeoReport V2 API key
  --html-url-template=HTML_URL_TEMPLATE
                        template for creating html urls for items based on
                        their identifiers, eg http://example.com/{id}.html
  --days-prior=DAYS_PRIOR
                        how many days ago to start scraping
  --schema=SCHEMA       slug of news item type to use
  --http-cache=HTTP_CACHE
                        location to use as an http cache.  If a cached value
                        is seen, no update is performed.
  --jurisdiction-id=JURISDICTION_ID
                        jurisdiction identifier to provide to api
  -v, --verbose         Verbose output.
  -q, --quiet           No output.


The scraper script is ``PATH/TO/ebdata/scrapers/general/open311/georeportv2.py``
and a suitable schema can be loaded by doing
``django-admin.py loaddata PATH/TO/ebdata/scrapers/general/open311/open311_service_requests_schema.json``.

(Tip: You can get an open311 endpoint for *any* location served by
seeclickfix, not just those listed on that page, by passing
``http://seeclickfix.com/<location-name>/open311/v2/``
as the API URL.)


SeeClickFix: scrapers.general.seeclickfix
-------------------------------------------------

A scraper for issues reported to `SeeClickFix <http://seeclickfix.com>`_.
Note you can also use the Open311 / GeoReport scraper described above,
since SeeClickFix supports the GeoReport API as well; we have both
scrapers because the SeeClickFix native API has been around longer.

Usage::

 seeclickfix_retrieval.py [options] city state
 
 Options:
  -h, --help     show this help message and exit
  -v, --verbose  Verbose output.
  -q, --quiet    No output.


The scraper script is ``PATH/TO/ebdata/scrapers/general/seeclickfix/seeclickfix_retrieval.py``
and a suitable schema can be loaded by doing
``django-admin.py loaddata PATH/TO/ebdata/scrapers/general/seeclickfix/seeclickfix_schema.json``.


ebdata.scrapers.us
------------------

Scrapers for specific city data sources in the USA. Currently this
includes only scrapers for Boston, MA:

* ebdata/scrapers/us/ma/boston/building_permits/
* ebdata/scrapers/us/ma/boston/businesses/
* ebdata/scrapers/us/ma/boston/events/
* ebdata/scrapers/us/ma/boston/police_reports/
* ebdata/scrapers/us/ma/boston/restaurants/

Many of these are used for http://demo.openblockproject.org.
For more information, see the source of each script.


ebdata Package
==============

:mod:`ebdata` Package
---------------------

.. automodule:: ebdata
    :members:
    :show-inheritance:

Subpackages
-----------

.. toctree::

    ebdata.blobs
    ebdata.geotagger
    ebdata.nlp
    ebdata.parsing
    ebdata.retrieval
    ebdata.scrapers
    ebdata.templatemaker
    ebdata.textmining
    ebdata.utils


.. _everyblock: https://github.com/openplans/openblock-extras/blob/master/docs/everyblock.rst
