:tocdepth: 1

.. sectnum::


**ABSTRACT**

This note covers two perspectives on accessing data in a Rubin middleware `Butler` from external code.

**Enabling end-user access to data extracted from a Butler**

There is a requirement to allow external code to run on Rubin images.
This requires at least accessing the images via butler and creating FITS files or such for an external program.
This note provides some background and a notebook approach to doing this.

**Running external executables under the control of a QuantumGraph pipeline**

Given data that are already in a Butler, there may be circumstances in which
it's desirable for users, or even in some cases, project staff, to run an
external executable on those data, and to do so under the control of a
Pipeline run via QuantumGraph.
This might be, for instance, because the natural way to control the iteration
through the input data is by traversing a collection.

Under some circumstances it may even be desirable to return outputs from
the external executable to the Butler.

The second part of this note therefore addresses the possible design of a PipelineTask
that serves as a wrapper for an external executable.


Enabling end-user access to data extracted from a Butler
========================================================

Introduction
------------

:cite:`DMTN-223` outlines plans for batch access to Rubin data.
This would be via the USDF and using the Rubin pipelines.
There is still  a requirement "DMS-REQ-0128" to allow "means for applying user-provided processing to image data".
This we feel could be met by demonstrating the possibility to extract FITS files from the butler repository and use a third party application on them.

This is demonstrated in the notebook in the repository of this technote called  `ExternalCode.ipynb <ExternalCode.ipynb>`_.

External Code
-------------

For this example we will use _usdf-rsp.slac.stanford.edu/nb: the USDF RSP Notebook Environment”.
The notebook environment allows a user to bring in their own or third party code.
For this exercise we will run sextractor over some Rubin images which will also be written to FITS.


sep
^^^

sep is an python version of sextractor.
This may be installed on the RSP by opening a terminal and executing
cmd:
pip install sep


Get some images
---------------

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
---------

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



Running external executables under the control of a QuantumGraph pipeline
=========================================================================

The second part of this note considers the related case of a desire to run a
pre-existing external executable on data contained in a Butler, and optional
return outputs from that to the Butler, under the control of a Pipeline/QuantumGraph
pipeline.

This note envisions meeting that need by providing a "wrapper" PipelineTask that
supports this use case, and is able to be used for simple cases purely via config
parameters.

For the initial version of this note, we define the problem with a set of requirements.
Later versions will discuss implementation.

Use cases
---------

The principal use case is to allow an external executable to be executed over a collection
of data specified via QuantumGraph-style mechanisms, with a single execution per input
dataset.

This version of the wrapper is not required to support use cases with multiple inputs of
the same type -- for example, image coaddition.

However, it is required to support executables which take a set of inputs of different types
or roles, as long as all of them can be identified by the same, single DataID.
For instance, it should support ISR-type use cases, where there is a single input image,
but ancillary data such as a bias frame or flat field are required.
Similarly, image subtraction use cases with a single-epoch image and a template image
could be supported, as long as the template image can be located using the single-epoch
image's DataID.

The reference use case envisions an executable which takes its inputs as POSIX files and
has a command-line syntax that is compatible with providing either relative or absolute
pathnames for each input.
In this case, if the input datasets in the input Butler are on an accessible Posix filesystem,
the execution should be able to be performed using absolute pathname references to the data
at rest.

If the input datasets are in an object store, the wrapper will copy them out to a temporary
Posix workspace and provide the executable with their temporary Posix pathnames.

It should be a configurable option to force the wrapper to always copy the input data to
a temporary space, to support use cases where the input data are in a Butler Posix
location that is technically writable by the executing process, and where for
data-safety reasons it's not desirable to give the external executable a pathname into
that directory.

If the external executable is capable of using s3: URIs, it should be an option to pass
them directly in.

Dealing with output
^^^^^^^^^^^^^^^^^^^

There are essentially three categories of use cases for output:

- Console output only

- Output files not needing to be entered into the Butler being used

- Output files intended to be written back into the Butler, as when the external executable
  is to be used as a stage in a longer, otherwise native PipelineTask pipeline

We assume that the wrapper creates a unique output directory for each DataID,
so that if the external executable writes out files with an unchanging name
(imagine "output.fits"), they don't stomp on each other.

For output files that don't need to end up back in the Butler, that directory
would be left around for the disposition of the user invoking the pipeline.

For output files that are supposed to end up back in the Butler, because this is by
hypothesis a 1:1 PipelineTask, the input DataID is also used for the output.
It is legitimate to have multiple output files as long as each one has a different
dataset type.
There are basically two options for making the actual entry into the Butler:

# Ingest-like treatment, where no output formatter is involved, and no put() call is made,
  taking the file as written by the external executable and just creating metadata for it
  in the Registry.
  For an object-store-based Butler this would in general involve copying the file from
  its temporary output location into the object store, under whatever name would otherwise
  be chosen based on its DataID and dataset type.
  For a Posix-filesystem-based Butler this could be done with "mv" so that if the Butler's
  backing filesystem happens to be the same as the one used for the temporary output directory,
  there is no unnecessary data copying performed.
  This approach does not guarantee that the resulting Butler dataset can actually be
  used via get().
  It allows the wrapper to avoid any dependency on the output dataset type.
# put()-style treatment, where, after the external executable has terminated, the wrapper
  reads the file into memory in some way, and then uses an ordinary put() to write it out,
  using a suitable formatter.
  This is less efficient but provides more guarantees.
  It requires the existence of the formatter and its accessibility to the wrapper.
  It's probably difficult to implement in a generic way, though with care the necessary
  user-specific code could be made easily pluggable into an otherwise generic wrapper.


Requirements
------------

("WPT" is meant to suggest "Wrapper PipelineTask")

Nature of the work to be executed
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

WPT-001
    The WPT shall be a subclass of PipelineTask.

WPT-002
    The WPT shall be capable of executing a single, user-specified Posix executable,
    as a subprocess of the Python interpreter, under the control of a QuantumGraph-style
    pipeline.

WPT-003
    A Pipeline may contain more than one WPT, and their executions shall not interfere
    with each other as long as the executable(s) wrapped do not have non-obvious side
    effects.

WPT-004
    A single call to runQuantum() of the WPT shall have a 1:1 relationship with an
    execution of the external executable.
    The WPT is not required to, for instance, work with the external executable as if
    a coroutine, supplying a sequence of input files to a persistently executing process.

WPT-005
    The inputs to a single execution of the WPT shall all be specifyable via a single DataID.
    The DataID shall resolve to a single dataset for each input DatasetType provided.

Access to input data
^^^^^^^^^^^^^^^^^^^^

WPT-101
    The WPT shall support as input data any dataset type which is persisted as a single file
    or S3 object.  The WPT is not required to support composite datasets.

WPT-102
    The WPT shall support execution of executables which require multiple inputs, as long as
    those inputs

WPT-102
    The WPT is not required to support external executables which make assumptions about
    the organization of multiple input files into directories.


.. rubric:: References
.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
    :style: lsst_aa
