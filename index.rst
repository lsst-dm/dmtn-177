..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   Having every pipeline task in every job in a workflow read from registry and datastore and then write to registry and datastore does not scale when millions of jobs are involved.
   Instead we need an approach where the reads and writes are load-balanced to provide sufficient scaling.
   This note describes the issues and provides a possible design.

Introduction
============

When pipelines are executed by invoking the ``pipetask`` command, each ``PipelineTask`` that is executed results in calls to ``Butler.get()`` at the start to retrieve the relevant data and calls to ``Butler.put()`` at the end to write the results to Butler registry.
These reads and writes are run sequentially with the writes using separate transactions for each dataset.
If multiple pipeline tasks are involved in a single execution the registry reads and writes happen between each one.
In a workflow environment each job being executed currently uses a shared PostgreSQL registry and a remote object store.
The general scheme is described in :cite:`DMTN-157` in the context of the Google Proof of Concept.
This approach works well until thousands of jobs are running simultaneously and they all need to connect to the database at the same time.
It's also made worse if jobs take roughly the same amount of time to execute and they are all starting at the same time and completing at the same time and can lead to large connection spikes to the registry, leading to dropped connections.
We mitigated some of this problem in the Google Proof of Concept by spreading the start time of the initial jobs randomly over a ten minute period.
This smoothed out some of the spikes and allowed us to execute a few thousand jobs.

Unfortunately this approach will not scale when hundreds of thousands to millions of jobs are in flight at the same time and all trying to write to the same PostgresSQL server.
It is also incompatible with a distributed processing approach where multiple data facilities are processing a fraction of the data and do not want to be reliant on one single server.
We therefore need to change how we manage access to the registry and, possibly, datastores when we execute large workflows.

In the discussion below workflow is defined as a single submission to a workflow execution system, generally via BPS, involving the conversion of a ``QuantumGraph`` into distinct jobs that are executed on separate nodes.
Each job can consist of one or more quanta where chaining of quanta can be used where all the outputs of one quantum are immediately used by the subsequent quantum.
Currently jobs consist of a single call to the ``pipetask`` command.

Assumptions
===========

The datasets created by the Rubin science pipelines are large and in a cloud environment it makes sense for the object store to be used directly by the Butler rather than attempting to use the workflow execution engine to stage the files to and from a workflow storage area to node local storage.
Object stores are already designed to be efficient under load and should not cause any bottlenecks.

In this document it is assumed that the workflow execution is run with the credentials of a service account that can do direct connections to the object store and cloud database.
The batch submission system would check that the user submitting the workflow has permissions to read from the input collections and write to the output collection.
Workflow executions using the standard infrastructure will never attempt to delete datasets from registry.
If jobs are being executed with science platform user credentials it will be necessary for the workflow execution system to use the client/server Butler interface (:cite:`DMTN-176`) which may result in an additional bottleneck to the registry and will require significant usage of signed URLs.

.. note::
  Could rate-limiting registry access be mediated by the Registry REST server to act as a queue?


Reducing Registry Access
========================

When running at scale the datastore reads and writes involving S3 or Google Cloud Storage are robust and load-balancing is handled natively by the cloud infrastructure.
With a single PostgreSQL registry where all jobs must see an up-to-date view of the registry, reducing connections to the database, the number of transactions running inside the database, and the length of those transactions is critical when attempting to run with hundreds of thousands of simultaneous jobs.

There are many different points at which we can decide to do the registry updates and they are described below.

Immediate
---------

The current approach used by the ``pipetask`` executor is to call ``butler.get()`` sequentially for each of the requested input datasets (sometimes this read can be deferred, or example when coadding many input images).
When the ``PipelineTask`` completes each of the output datasets is then stored in the butler with a ``butler.put()``.

It will use the Butler it has been configured to use and in the batch system that is a shared PostgreSQL registry and S3-like object store.

One problem with this approach is that the Butler is designed such that the registry changes and the file writing must all be completed within a database transaction.
This can result in the database transactions taking a relatively long time for large data files (which first have to be written to local scratch space and then uploaded to the object store).
Long-lived transactions can cause trouble with database responsiveness when tens of thousands of connections are active.

There are also concerns, as expressed in :jira:`DM-26302`, that the connection management in Butler registry is non-optimal since it disables connection pooling in SQLAlchemy.

.. note::
  How long does the ``pipetask`` executor keep each registry connection open?
  Is a single connection used for the duration of execution of each quanta?
  Or is there one connection used for all the gets at the start and a new connection used for all the puts at the end?

Job-level Pooling
-----------------

One approach to minimizing registry interactions is to use a local registry during the pipeline execution job and then to batch the registry writes once the pipeline(s) associated with that job finishes.

In this scenario the job would start by exporting the relevant subset of the global registry to a local file (in YAML or JSON format) and importing it into a newly-created local SQLite-based Butler.
This SQLite database would still refer to the datasets in their original datastore location.
The pipeline would then be executed using the local registry and would interact as before with the cloud object store.
When the processing completes the newly-created records would be exported from the local registry and imported to the cloud registry.
This would use the automatic transfer mode since the files are already present in the object store in the correct place.

If quanta are chained, with the outputs of one ``PipelineTask`` feeding directly to the inputs of the next within the same job, the datastore will be configured to do local caching and write the file locally as well as to the object store so that the next one would read the file directly.

Doing this would have the advantage that the object store writes are no longer linked to a long-lived transaction in the shared database.
On the other hand, if the pipeline processing fails in some way it is likely that there will be files in the object store that are not present in the cloud registry.
These files can be deleted in a clean up job or they can be ignored, allowing them to be over written.
Regardless, if the node is pre-empted during processing after files have been written, the orphaned files will be present and there will be no associated registry entries so attempting to clean-up does not gain us anything other than saving storage space if that run is never used again.
The execution system is already capable of over-writing an existing file if it is not known to registry and datastore ensures that the file name is unique for a specific dataId and run combination.

If pre-emption occurs during the final registry import it is assumed that there is a single transaction that will be completely rolled back.
The end result would be that the entire processing would have to be redone.
If the pre-emption occurs in the short time between completion of the registry import and the completion of the job, we should ensure that the ``pipetask`` command is configured to complete without action when the job is restarted.

Externalized
------------

The most flexible approach is for the workflow generator to insert extra jobs into the workflow that handle registry merging and registry syncing.

A new job would be inserted before the initial pipeline jobs in the graph that downloads the initial registry state from the cloud registry and stores it in a SQLite file.
This SQLite file would then be provided as an input to the first job.
The pipeline job would use that SQLite registry and a GCS-with-local-cache datastore as described in the previous section.
On completion the output collected by the workflow execution system would be an updated SQLite registry.
If the next pipeline job only takes files from a single input job this SQLite file would be passed directly to the next pipeline job.
If, on the other hand, the next job gathers files from multiple jobs, the workflow generator will insert a new job before it that reads all the SQLite files and merges them into a single output for that job.
It can optionally also sync registry information to the cloud registry and trim the SQLite file that it passes on.
This sync and prune could be a distinct job that can be inserted after the merge job in the graph.
At the end node (or nodes) of each workflow graph a final job will be inserted that updates the cloud registry with the final state.

In this scenario pre-emption has no impact on registry state for pipeline jobs since they are running with an entirely local SQLite registry.
Pre-emption still has to be understood in terms of the registry sync jobs and requires that the remote update of the cloud registry happens in a single transaction.

This approach allows the submission system to decide whether the registry is updated multiple times within the graph or solely at the end since the registry merging jobs can be configured to either merge the inputs and pass them on complete, or merge, sync and trim.
A trimmed registry can not be passed on to the next job if the remainder has not been synced with the cloud registry.

Summary
=======

Both the job-level pooling and fully externalized approach to Registry should be able to improve throughput significantly.
The job-level pooling approach is simplest to implement and should be attempted first and performance compared to the current system.

.. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
    :style: lsst_aa
