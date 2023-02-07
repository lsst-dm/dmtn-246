:tocdepth: 1

.. sectnum::


Abstract
========

There is a requirement to allow external code to run on Rubin images. This requires at least accessing the images via butler and creating FITS files or such for an external program. This note provides some background and a notebook approach to doing this.

Introduction
============

:cite:`DMTN-223` outlines plans for batch access to Rubin data. 
This would be via the USDF and using the Rubin pipelines.
There is still  a requirement "DMS-REQ-0128" to allow "means for applying user-provided processing to image data". 
This we feel could be met by demonstrating the possibility to extract FITS files from the butler repository and use a third party application on them. 

This is demonstrated in the notebook in the `repository of this technote  <ExternalCode.ipynb>`_. 

External Code
=============
For this example we will use _usdf-rsp.slac.stanford.edu/nb: the USDF RSP Notebook Environment”.
The notebook environment allows a user to bring in their own or third party code. 
For this exercise we will run sextractor over some Rubin images which will also be written to FITS. 


sep
---
sep is an python version of sextractor. 
This may be installed on the RSP by opening a terminal and executing
cmd:
pip install sep



Get some images
=============== 
It is assumed the reader will read tutorials on butler to find images.
There are tutorials on the `DP0.2 tutorial site`_.

The notebook uses butler to get all the exposures from 1 visit from the DC2 simulation at the USDF.
It writes out 20 of them to FITS.
This is an arbitrary number to just not use all the quota. 

.. code-block:: python

    datasetType = 'calexp'
    dataId = {'visit': 733724}
    datasetRefs = set(registry.queryDatasets(datasetType, dataId=dataId))

    for count, exp in enumerate(datasetRefs):
        fn=f"Rubin-calexp-{exp.dataId['visit']}-{exp.dataId['detector']}.fits"
        calexp = butler.get('calexp',exp.dataId)
        calexp.writeFits(fn)
        if (count > 20):
            break


Apply sep
========= 
Using a basic sep  we can open each image, calculate the background and extract sources. 
These are aggregated and written to catalog.csv.


.. code-block:: python

	import glob
	from astropy.io import fits
	import sep
	import csv
	filelist = glob.glob('Rubin-calexp*.fits')
	catfile = 'catalog.csv'
	outfile = open(catfile,'w')
	catalog = csv.writer(outfile,delimiter=',')
	ocount = 0
		       
	for ffile in filelist:
	    hdul = fits.open(ffile)  
	    data = hdul[1].data.byteswap().newbyteorder()  # sep wants this 
	    bkg = sep.Background(data)
	    # subtract the background
	    data_sub = data - bkg
	    objects = sep.extract(data_sub, 1.5, err=bkg.globalrms)
	    for o in objects:
		catalog.writerow(o)
	    ocount = ocount + len(objects)
	outfile.close()

Running this should get something like 30K objects from 22 images.

.. _DP0.2 tutorial site: https://dp0-2.lsst.io/tutorials-examples/index.html

.. rubric:: References
.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
    :style: lsst_aa
