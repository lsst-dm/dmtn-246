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

:cite:`DMTN-223` outlines plans for batch access to Rubin data. 
This would be via the USDF and using the Rubin pipelines.
There is still  a requirement "DMS-REQ-0128" to allow "means for applying user-provided processing to image data". 
This we feel could be met by demonstrating the possibility to extract FITS files from the butler repository and use a third party application on them. 

This is demonstrated in the notebook in the repository of this technote :file:`ExternalCode.ipynb`. 

External code
=============
The notebook environment allows a user to bring in their own or third party code. 
For this excercise we will use sextractor. 
This has a conda install. 
Having logged in to _usdf-rsp.slac.stanford.edu/nb: the USDF RSP Notebook Enviroment" and opening a new terminal type:

:cmd:  conda install -c conda-forge astromatic-source-extractor


.. rubric:: References
.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
    :style: lsst_aa
