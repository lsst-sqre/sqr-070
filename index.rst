:tocdepth: 1

.. sectnum::

.. Metadata such as the title, authors, and description are set in metadata.yaml

.. TODO: Delete the note below before merging new content to the main branch.

.. note::

   **This technote is a work-in-progress.**

Abstract
========

The Influx-supplied telegraf-operator is a good concept, but it doesn't work very well for Rubin's needs.  

While we like the idea of specifying Prometheus metrics endpoints to scrape via annotations, doing it at the pod level does not work well for us.  Since Phalanx has the concept of applications, which generally map one-to-one to namespaces, we would prefer an annotation system where the annotations specifying metrics endpoints were attached to the application object (or the namespace) rather than pod objects.  This technote will propose an annotation syntax that would enable endpoint specification that fits our use-case, and guide an implementation of an operator to realize that specification.

Add content here
================

Add content here.
See the `reStructuredText Style Guide <https://developer.lsst.io/restructuredtext/style.html>`__ to learn how to create sections, links, images, tables, equations, and more.

.. Make in-text citations with: :cite:`bibkey`.
.. Uncomment to use citations
.. .. rubric:: References
.. 
.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
