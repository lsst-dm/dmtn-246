:tocdepth: 1

.. sectnum::

.. Metadata such as the title, authors, and description are set in metadata.yaml

.. TODO: Delete the note below before merging new content to the main branch.

.. note::

   **This technote is a work-in-progress.**

Abstract
========

There is a requirement to allow external code to run on Rubin images. This requires at least accessing the images via butler and creating FITS files or such for an external program. This note provides some background and a notebook approach to doing this.

Introduction
============

:cite:`dmtn-223` outlines plans for batch access to Rubin data. 
This would be via the USDF and using the Rubin pipelines.
Threre is still  a requirement "DMS-REQ-0128" to allow "means for applying user-provided processing to image data". 
This we feel could be met by deomstrating the possibility to extract FITS files from the butler repository and use a third party applicaiton on them. 

This is demostrated in the notebook in the repository of this technote :file:`. 


.. rubric:: References
.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
    :style: lsst_aa
