.. image:: https://img.shields.io/badge/dmtn--177-lsst.io-brightgreen.svg
   :target: https://dmtn-177.lsst.io/
.. image:: https://github.com/lsst-dm/dmtn-177/workflows/CI/badge.svg
   :target: https://github.com/lsst-dm/dmtn-177/actions/

##################################################
Limiting Registry Access During Workflow Execution
##################################################

DMTN-177
========

Having every job in a workflow read from registry and datastore and then write to registry and datastore does not scale when millions of jobs are involved. Instead we need an approach where the reads and writes are load-balanced to provide sufficient scaling. This note describes the issues and provides a possible design.

**Links:**

- Publication URL: https://dmtn-177.lsst.io/
- Alternative editions: https://dmtn-177.lsst.io/v
- GitHub repository: https://github.com/lsst-dm/dmtn-177
- Build system: https://github.com/lsst-dm/dmtn-177/actions/

Build this technical note
=========================

You can clone this repository and build the technote locally if your system has Python 3.12 or later:

.. code-block:: bash

   git clone https://github.com/lsst-dm/dmtn-177
   cd dmtn-177
   make init
   make html

Repeat the ``make html`` command to rebuild the technote after making changes.
If you need to delete any intermediate files for a clean build, run ``make clean``.

The built technote is located at ``_build/html/index.html``.

Publishing changes to the web
=============================

This technote is published to https://dmtn-177.lsst.io/ whenever you push changes to the ``main`` branch on GitHub.
When you push changes to a another branch, a preview of the technote is published to https://dmtn-177.lsst.io/v.

Editing this technical note
===========================

The main content of this technote is in ``index.rst`` (a reStructuredText file).
Metadata and configuration is in the ``technote.toml`` file.
For guidance on creating content and information about specifying metadata and configuration, see the Documenteer documentation: https://documenteer.lsst.io/technotes.
