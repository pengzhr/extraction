==========
Extraction
==========

Extraction is a Python package for extracting titles, descriptions,
images and canonical urls from web pages. You might want to use Extraction
if you're building a link aggregator where users submit links and you
want to display them (like submitting a link to Facebook, Digg or Delicious).

Extraction is not a web crawling or content retrieval mechanism, rather
it is a tool to use on data which has always been retrieved or crawled
by a different tool.

Developed against Python 2.7, but should also work with Python 2.6+ if
you install the importlib requirement.


Hello World Usage
=================

An extremely simple example of using `extraction` is::

    >>> import extraction
    >>> import requests
    >>> url = "http://lethain.com/social-hierarchies-in-engineering-organizations/"
    >>> html = requests.get(url).text
    >>> extracted = extraction.Extractor().extract(html, source_url=url)
    >>> extracted.title
    >>> "Social Hierarchies in Engineering Organizations - Irrational Exuberance"
    >>> print extracted.title, extracted.description, extracted.image, extracted.url

Note that `source_url` is optional in extract, but is recommended
as it makes it possible to rewrite relative urls and image urls
into absolute paths. `source_url` is not used for fetching data,
but can be used for targetting extraction techniques to the correct
domain.

More details usage examples, including how to add your own
extraction mechanisms, are beneath the installation section.


Installation
============

The simplest way to install Extraction is via PyPi::

    pip install extraction

If you want to develop against extraction, you can install from GitHub::

    git clone
    cd extraction
    virtualenv env
    . ./env/bin/activate
    pip install -r requirements.txt
    python setup.py develop

Then you can run the tests::

    python tests/tests.py

All of which should pass in a sane installation.


Extraction Techniques
=====================

This section lists the current techniques used by extraction.
To rerank the techniques, remove techniques or add new techniques
of your own, look at the `Using Extraction` section below.


extraction.techniques.FacebookOpengraphTags
-------------------------------------------

For better or for worse, the highest quality source of page data is usually
the `Facebook Opengraph meta tags <https://developers.facebook.com/docs/opengraphprotocol/>`.
This technique uses Opengraph tags, which look like this::

    <head>
        ...
        <meta property="og:title" content="Something"/>
        <meta property="og:url" content="http://www.example.org/something//"/>
        <meta property="og:image" content="http://images.example.org/a/"/>
        <meta property="og:description" content="Something amazing."/>
        ...
    </head>

as their source of data.


Using Extraction
================

This section covers various ways to use extraction, both using
the existing extraction techniques as well as add your own.

For more examples, please look in the `extraction/examples`
directory.


Basic Usage
-----------

The simplest possible example is the "Hello World" example from above::

    >>> import extraction
    >>> import requests
    >>> url = "http://lethain.com/social-hierarchies-in-engineering-organizations/"
    >>> html = requests.get(url).text
    >>> extracted = extraction.Extractor().extract(html, source_url=url)
    >>> extracted.title
    >>> "Social Hierarchies in Engineering Organizations - Irrational Exuberance"
    >>> print extracted.title, extracted.description, extracted.image, extracted.url


Using Custom Techniques and Changing Technique Ordering
-------------------------------------------------------

The order techniques are run in is significant, and the most accurate
techniques should always run first, and more general, lower quality
techniques later on.

This is because titles, descriptions, images and urls are stored
internally in a list, which is built up as techniques are run,
and the `title`, `url`, `image` and `description` properties
simply return the first item from the corresponding list.

Techniques are represented by a string with the full path to the
technique, including its class. For example `"extraction.technique.FacebookOpengraphTags"`
is a valid representation of a technique.

The default ordering of techniques is within the  extraction.Extractor's
`techniques` class variable, and is::

    extraction.techniques.FacebookOpengraphTags

You can modify the order and inclusion of techniques in three ways.
First, you can modify it by passing in a list of techniques to the
optional `techniques` parameter when initializing an extraction.Extractor::

    >>> techniques = ["my_module.MyTechnique", "extraction.techniques.FacebookOpengraphTags"]
    >>> extractor = extraction.Extractor(techniques=techniques)

The second approach is to subclass Extractor with a different value of `techniques`::

    from extraction import Extractor

    class MyExtractor(Extractor):
        techniques = ["my_module.MyTechnique"]

Finally, the third option is to directly modify the `techniques` class variable.
This is probably the most unpredictable technique, as it's possible for mutiple
pieces of code to perform this modification and to create havoc, if possible
use one of the previous two techniques to avoid future debugging::

    >>> import extraction
    >>> extraction.Extractor.techniques.insert(0, "my_module.MyAwesomeTechnique")
    >>> extraction.Extractor.techniques.append("my_module.MyLastReportTechnique")

Again, please try the first two techniques instead if you value sanity.


Writing New Technique
---------------------

It may be that you're frequently parsing a given website and
aren't impressed with how the default extraction techniques are
performing. In that case, consider writng your own technique.

Let's take for example a blog entry at `lethain.com <http://lethain.com/social-hierarchies-in-engineering-organizations/>`,
which uses the `H1` tag to represent the overall blogs title,
and always uses the first `H2` tag in `DIV.page` for its actual
title.

A technique to properly extract this data would look like::

    from extraction.techniques import Technique
    from bs4 import BeautifulSoup
    class LethainComTechnique(Technique):
        def extract(self, html):
            "Extract data from lethain.com."
            soup = BeautifulSoup(html)
            page_div = soup.find('div', class_='page')
            text_div = soup.find('div', class_='text')
            return { 'titles': [page_div.find('h2').string],
                     'dates': [page_div.find('span', class_='date').string],
                     'descriptions': [" ".join(text_div.find('p').strings)],
                     'tags': [x.find('a').string for x in page_div.find_all('span', class_='tag')],
                     'images': [x.attrs['src'] for x in text_div.find_all('img')],
                     }

To integrate your technique, take a look at the `Using Custom Techniques and Changing Technique Ordering`
section above.

Adding new techniques incorporating microformats is an interesting
area for some consideration. Most microformats have very limited
usage, but where they are in use they tend to be high quality sources
of information.


Subclassing Extracted to Extract New Types of Data
--------------------------------------------------

Your techniques can return non-standard keys in the dictionary
returned by `extract`, which will be available in the `Extracted()._unexpected_values`
dictionary. In this way you could fairly easily add support for extracting
addresses or whatnot.

For a contrived example, we'll extract my address from `willarson.com <http://willarson.com/>`,
which is in no way a realistic example of extracting an address, and is
only meant as an example of how to add a new type of extracted data.

As such, to add support for extracting address should look like (a fuller,
commented version of this example is available in `extraction/examples/new_return_type.py`,
I've written this as concisely as possible to fit into this doc more cleanly)::

    from extraction.techniques import Technique
    from extraction import Extractor, Extracted
    from bs4 import BeautifulSoup

    class AddressExtracted(Extracted):
        def __init__(self, addresses=None, *args, **kwargs):
            self.addresses = addresses or []
            super(AddressExtracted, self).__init__(*args, **kwargs)

        @property
        def address(self):
            return self.addresses[0] if self.addresses else None

    class AddressExtractor(Extractor):
        "Extractor which supports addresses as first-class data."
        extracted_class = AddressExtracted
        text_types = ["titles", "descriptions", "addresses"]

    class AddressTechnique(Technique):
        def extract(self, html):
            "Extract address data from willarson.com."
            soup = BeautifulSoup(html)
            return {'addresses': [" ".join(soup.find('div', id='address').strings)]}

Usage would then look like::

    >>> import requests
    >>> from extraction.examples.new_return_type import AddressExtractor
    >>> extractor = AddressExtractor()
    >>> extractor.techniques = ["extraction.examples.new_return_type.AddressTechnique"]
    >>> extracted = extractor.extract(requests.get("http://willarson.com/"))
    >>> extracted.address
    "Cole Valey San Francisco, CA USA"

There you have it, extracted addresses as first class extracted data.


Passing Parameters to Techniques
--------------------------------

There isn't a mechanism for passing parameters to Techniques
when they are initialized, but it is possible to customize
the behavior of Techniques in a couple of ways.

First, you can simply subclass the Technique with the specific
behavior you want, perhaps pulling the data from Django settings
or what not.

    class MyTechnique(Technique):
        def __init__(self, *args, **kwargs):
            if 'something' in kwargs:
                self.something = kwargs['something']
	        del kwargs['something']
            else:
                self.something = "something else"
            return super(MyTechnique, self).__init__(*args, **kwargs)

        def extract(html, source_url=None):
            print self.something
            return super(MyTechnique, self).extract(html, source_url=source_url)

Second, all techniques are passed in the Extractor being used
to process them, so you can bake the customization into an
extraction.Extractor subclass::

    from extraction import Extractor
    from extraction.techniques import Technique

    class MyExtractor(Extractor):
        techniques = ["my_module.MyTechnique"]
        def __init__(self, something, *args, **kwargs):
            self.something = something
            super(MyExtractor, self).__init__(*args, **kwargs)

    class MyTechnique(Technique):
        class extract(self, html, source_url=None):
            print self.extractor.something
            return super(MyTechnique, self).extract(html, source_url=source_url)

Between these two techniques, it should be feasible to get the
customization of behavior you need.


Implementation Details
======================

I've tried to comment the classes and modules themselves in a fairly
indepth fashion, and would recommend reading them for the most details,
the recommended reading order is::

    extraction/tests.py
    extraction/__init__.py
    extraction/techniques.py

Hopefully all questions are answered therein.


Contributions, Questions, Concerns
==================================

Please open a GitHub pull-request with any improvements,
preferably with tests, and I'll be glad to merge it in.
