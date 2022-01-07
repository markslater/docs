.. _minio-bucket-replication-resynchronize:


========================================
Resynchronize Bucket from Remote Replica
========================================

.. default-domain:: minio

.. contents:: Table of Contents
   :local:
   :depth: 1

The procedure on this page resynchronizes the contents of a MinIO
bucket using a healthy replication remote. Resynchronization supports
recovery after partial or total loss of data on a MinIO deployment in a 
replica configuration.

For example, consider a MinIO active-active replication configuration similar
to the following:

.. image:: /images/replication/active-active-twoway-replication.svg
   :width: 600px
   :alt: Active-Active Replication synchronizes data between two remote deployments.
   :align: center

Resynchronization allows using the healthy data on one of the participating
MinIO deployments as the source for rebuilding the other deployment.

Resynchronization is a per-bucket process. You must repeat resynchronization
for each bucket on the remote which suffered partial or total data loss.

.. _minio-bucket-replication-serverside-resynchronize-requirements:

Requirements
------------

MinIO Deployments Must Be Online
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Resynchronization requires both the source and remote deployments be online and
able to accept read and write operations. The source *must* have 
complete network connectivity to the remote.

The remote deployment may be "unhealthy" in that it has suffered partial or
total data loss. Resynchronization addresses the data loss as long as both
source and destination maintain connectivity.

Disable Replication on Remote Bucket
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This procedure assumes the remote bucket has no existing replication
configuration rules *or* that all rules are disabled. This ensures no unexpected
or abnormal replication behavior during resynchronization.

This procedure includes steps for configuring replication on the remote bucket
once resynchronization completes.

Replication Requires MinIO Deployments
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

MinIO server-side replication only works between MinIO deployments. Both
the source and destination deployments *must* run MinIO.

Replication Requires Versioning
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

MinIO relies on the immutability protections provided by versioning to
synchronize objects between the source and replication target.

Use the :mc-cmd:`mc version enable` command to enable versioning on 
*both* the source and destination bucket before starting this procedure:

.. code-block:: shell
   :class: copyable

   mc version enable ALIAS/PATH

- Replace :mc-cmd:`ALIAS <mc version ALIAS>` with the
  :mc:`alias <mc alias>` of the MinIO cluster.

- Replace :mc-cmd:`PATH <mc version ALIAS>` with the bucket on which
  to enable versioning.

.. _minio-bucket-replication-serverside-resynchronize-permissions:

Required Permissions
~~~~~~~~~~~~~~~~~~~~

Bucket replication requires specific permissions on the source and
destination deployments to configure and enable replication rules. 

.. tab-set::

   .. tab-item:: Replication Admin

      The following policy provides permissions for configuring and enabling
      replication on a cluster. 

      .. literalinclude:: /extra/examples/ReplicationAdminPolicy.json
         :class: copyable
         :language: json

      - The ``"EnableRemoteBucketConfiguration"`` statement grants permission
        for creating a remote target for supporting replication.

      - The ``"EnableReplicationRuleConfiguration"`` statement grants permission
        for creating replication rules on a bucket. The ``"arn:aws:s3:::*``
        resource applies the replication permissions to *any* bucket on the
        source cluster. You can restrict the user policy to specific buckets
        as-needed.

      Use the :mc-cmd:`mc admin policy add` to add this policy to *both*
      deployments. You can then create a user on both deployments using
      :mc-cmd:`mc admin user add` and associate the policy to those users
      with :mc-cmd:`mc admin policy set`.

   .. tab-item:: Replication Remote User

      The following policy provides permissions for enabling synchronization of
      replicated data *into* the cluster. Use the :mc-cmd:`mc admin policy add`
      to add this policy to *both* deployments.

      .. literalinclude:: /extra/examples/ReplicationRemoteUserPolicy.json
         :class: copyable
         :language: json

      - The ``"EnableReplicationOnBucket"`` statement grants permission for 
        a remote target to retrieve bucket-level configuration for supporting
        replication operations on *all* buckets in the MinIO cluster. To
        restrict the policy to specific buckets, specify those buckets as an
        element in the ``Resource`` array similar to
        ``"arn:aws:s3:::bucketName"``.

      - The ``"EnableReplicatingDataIntoBucket"`` statement grants permission
        for a remote target to synchronize data into *any* bucket in the MinIO
        cluster. To restrict the policy to specific buckets, specify those 
        buckets as an element in the ``Resource`` array similar to 
        ``"arn:aws:s3:::bucketName/*"``.

      Use the :mc-cmd:`mc admin policy add` to add this policy to *both*
      deployments. You can then create a user on both deployments using
      :mc-cmd:`mc admin user add` and associate the policy to those users
      with :mc-cmd:`mc admin policy set`.

MinIO strongly recommends creating users specifically for supporting 
bucket replication operations. See 
:mc:`mc admin user` and :mc:`mc admin policy` for more complete
documentation on adding users and policies to a MinIO cluster.

Existing Object Replication
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Resynchronization requires :ref:`existing object replication
<minio-replication-behavior-existing-objects>`. Specifically, resynchronization
only applies to those replication configuration rules created where
:mc-cmd-option:`~mc replicate add replicate` includes ``"existing-objects"``.

Use :mc-cmd:`mc replicate ls` to list the replication rules for a bucket and
verify which rules have existing object replication enabled.

Considerations
--------------

Resynchronization Requires Time
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Resynchronization is a background processes that continually checks objects in
the source MinIO bucket and copies them to the remote as-needed. The time
required for replication to complete may vary depending on the number and size
of objects, the throughput to the remote MinIO deployment, and the load on the
source MinIO deployment. Total time for completion is generally not predictable
due to these variables.

MinIO recommends configuring load balancers or proxies to direct traffic only
to the healthy cluster until synchronization completes. The following commands
can provide insight into the resynchronization status:

- :mc-cmd:`mc replicate status` on the source and remote to track total 
  replicated data.

- Run ``mc ls -r --versions ALIAS/BUCKET | wc -l`` against both source and
  remote to validate the total number of objects and object versions on each.

Replication of Encrypted Objects
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

MinIO supports replicating objects encrypted with automatic Server-Side
Encryption (SSE-S3 or SSE-KMS). Both the source and destination buckets *must*
have automatic SSE-S3/SSE-KMS enabled for MinIO to replicate an encrypted
object.

As part of the replication process, MinIO *decrypts* the object on the source
bucket and transmits the unencrypted object. The destination MinIO cluster then
re-encrypts the object using the destination bucket SSE-S3 configuration. MinIO
*strongly recommends* :ref:`enabling TLS <minio-TLS>` on both source and
destination deployments to ensure the safety of objects during transmission.

MinIO does *not* support replicating client-side encrypted objects 
(SSE-C).

Replication of Locked Objects
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

MinIO supports replicating objects held under 
:ref:`WORM Locking <minio-object-locking>`. Both replication buckets *must* have
object locking enabled for MinIO to replicate the locked object. For
active-active configuration, MinIO recommends using the *same* 
retention rules on both buckets to ensure consistent behavior across
sites.

You must enable object locking during bucket creation as per S3 behavior. 
You can then configure object retention rules at any time.
Object locking requires :ref:`versioning <minio-bucket-versioning>` and
enables the feature implicitly.

Resynchronize Objects after Partial Data Loss
---------------------------------------------

This procedure assumes a MinIO deployment where a bucket participating in 
:ref:`active-active replication <minio-bucket-replication-serverside-twoway>`
or :ref:`multi-site replication <minio-bucket-replication-serverside-multi>`
suffers partial data loss. Specifically, all of the following should be true:

- The deployment maintains connectivity to the other MinIO deployments in the
  replication configuration.
- The bucket retained it's replication configuration settings.
- The bucket can successfully replicate new objects or write operations
  according to the configured replication rules.

The following steps use ``Alpha`` and ``Baker`` as placeholder :ref:`aliases
<alias>` for each MinIO deployment. ``Alpha`` provides the "healthy" source
data for the "unhealthy" ``Baker`` deployment.

Replace these example aliases with those of the MinIO deployments which act as
the healthy source and unhealthy remote. This procedure assumes access to the
necessary users and permissions to perform replication-related operations. See
:ref:`minio-users` and :ref:`MinIO Policy Based Access Control <minio-policy>`
for more complete documentation on MinIO users and policies respectively.

1) Identify User Accounts used for Replication
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

MinIO requires specific :ref:`permissions 
<minio-bucket-replication-serverside-resynchronize-permissions>` to support
replication. 

Use the :mc-cmd:`mc admin user list`, :mc-cmd:`mc admin user info` and
:mc-cmd:`mc admin policy info` to determine which users on both the 
healthy source and the unhealthy remote MinIO deployment support replication.

The remaining steps in this procedure assume that both deployments have
``ReplicationAdmin`` and ``ReplicationRemoteUser`` users with permissions
to replicate and receive replicated traffic respectively.

2) Configure ``mc`` Replication Admin Access
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Use :mc-cmd:`mc alias list` to verify the configured aliases on your host
machine. You can skip this step if you already have aliases for both
deployments which grant access to replication administration.

Use the :mc-cmd:`mc alias set` command to add a replication-specific alias for
both deployments:

.. code-block:: shell
   :class: copyable

   mc alias set AlphaReplication HOSTNAME AlphaReplicationAdmin LongRandomSecretKey
   mc alias set BakerReplication HOSTNAME BakerReplicationAdmin LongRandomSecretKey

3) Disable Replication on the Bucket which Requires Resynchronization
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

MinIO recommends disabling replication on the unhealthy deployment to avoid
unexpected or undesired replication states.

Configure the reverse proxy, load balancer, or other network control plane
managing connections to your active-active sites to stop sending traffic to the
unhealthy deployment.

The following command uses :mc-cmd:`mc replicate ls` to identify all replication
rules on the bucket. It then uses the :mc-cmd:`mc replicate edit` command to
disable the replication rules on the unhealthy bucket for the duration of the
resynchronization procedure. 

.. code-block:: shell
   :class: copyable

   mc replicate ls Baker/BUCKET

   mc replicate edit --id ID --state disable

- Replace ``BUCKET`` with the name of the bucket which requires 
  resynchronization.
- Replace the ``ID`` with the :mc-cmd:`~mc replicate edit id` of each
  replication rule to disable.

4) Start the Resynchronization Procedure
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Run the :mc-cmd:`mc admin bucket remote ls` command to list the configured
remote targets on the ``Alpha`` deployment for the ``BUCKET``:

.. code-block:: shell
   :class: copyable

   mc admin bucket remote ls Alpha/BUCKET

Identify the ARN associated to the ``BUCKET`` on the ``Baker`` deployment.
Run the :mc-cmd:`mc replicate resync` command to begin the resynchronization
process using this ARN value:

.. code-block:: shell
   :class: copyable

   mc replicate resync --remote-bucket "arn:minio:replication::UUID:BUCKET" alpha/BUCKET

- Replace the ``--remote-bucket`` value with the ARN of the ``BUCKET`` on the
  ``Baker`` deployment. 

  Use :mc-cmd:`mc admin bucket remote ls` to retrieve the replication target
  ARN.

- Replace the ``BUCKET`` with the name of the bucket on the source MinIO
  deployment.

The command returns a resynchronization job ID indicating that the process has
begun.

5) Monitor Resynchronization
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Use the :mc-cmd:`mc replicate status` command on the ``Baker`` deployment to
track the received replication data:

.. code-block:: shell
   :class: copyable

   mc replicate status Baker/BUCKET

The replication received bytes increases throughout the resynchronization 
process up through completion.

6) Re-enable Replication on the Resynchronized Bucket
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The following command uses :mc-cmd:`mc replicate ls` to identify all replication
rules on the bucket. It then uses the :mc-cmd:`mc replicate edit` command to
enable the replication rules on the bucket.

.. code-block:: shell
   :class: copyable

   mc replicate ls Baker/BUCKET

   mc replicate edit --id ID --state enable

- Replace ``BUCKET`` with the name of the bucket which requires 
  resynchronization.
- Replace the ``ID`` with the :mc-cmd:`~mc replicate edit id` of each
  replication rule to disable.

Perform basic validation that active replication to the the other sites in the
replication configuration succeeds. For example, create one or more test files
and check that they replicate as expected.

Once replication is validated and all sites are synchronized,configure the
reverse proxy, load balancer, or other network control plane managing
connections to your active-active sites to resume sending traffic to the
resynchronized deployment.

Resynchronize Objects after Total Data Loss
-------------------------------------------

Procedure
---------

1) Configure User Accounts and Policies for Replication
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The following code creates the user and policies necessary for
resynchronization of data from ``Alpha`` to ``Baker``. Replace the password
``LongRandomSecretKey`` with a long, random, and secure secret key as per
your organizations best practices for password generation.

.. code-block:: shell
   :class: copyable

   wget -O - https://docs.min.io/minio/baremetal/examples/ReplicationAdminPolicy.json | \
   mc admin policy add Alpha ReplicationAdminPolicy /dev/stdin
   mc admin user add Alpha alphaReplicationAdmin LongRandomSecretKey
   mc admin policy set Alpha ReplicationAdminPolicy user=alphaReplicationAdmin

   wget -O - https://docs.min.io/minio/baremetal/examples/ReplicationRemoteUserPolicy.json | \
   mc admin policy add Baker ReplicationRemoteUserPolicy /dev/stdin
   mc admin user add Baker bakerReplicationRemoteUser LongRandomSecretKey
   mc admin policy set Baker ReplicationRemoteUserPolicy user=bakerReplicationRemoteUser

You can skip this step if
both deployments already have users with the necessary :ref:`permissions
<minio-bucket-replication-serverside-resynchronize-permissions>`.

2) Configure ``mc`` Replication Admin Access
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Use the :mc-cmd:`mc alias set` command to add a replication-specific alias for
the ``Alpha`` deployment:

.. code-block:: shell
   :class: copyable

   mc alias set AlphaReplication HOSTNAME AlphaReplicationAdmin LongRandomSecretKey

3) Start the Resynchronization Procedure
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Run the :mc-cmd:`mc replicate resync` command to begin the resynchronization
process:

.. code-block:: shell
   :class: copyable

   mc replicate resync --remote-bucket "arn:minio:replication::UUID:BUCKET" alpha/BUCKET

- Replace the ``--remote-bucket`` value with the ARN of the ``BUCKET`` on the
  ``Baker`` deployment. 

  Use :mc-cmd:`mc admin bucket remote ls` to retrieve the replication target
  ARN.

- Replace the ``BUCKET`` with the name of the bucket on the source MinIO
  deployment.

The command returns a resynchronization job ID indicating that the process has
begun.

4) Monitor Resynchronization
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Use any of the following methods for tracking resynchronization progress:

- Run :mc-cmd:`mc replicate status` on both ``Alpha`` and ``Baker`` to track
  the sent and received replication data.

- Run :mc-cmd:`mc ls -r --versions ALIAS | wc -l <mc ls>` on
  ``Alpha`` and ``Baker`` to compare the total objects on both deployments.

5) Reconfigure Replication on Remote
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Once the remote bucket fully synchronizes with the source, you can safely
re-enable replication on the remote.

If you disabled replication rules on the remote bucket, use
:mc-cmd:`mc replicate edit` to re-enable those rules

If the remote bucket data loss included the replica configuration, use the
:mc-cmd:`mc admin bucket remote add` and :mc-cmd:`mc replicate add` commands
to rebuild the replication configuration to match the source.

For deployments which regularly back up their replication configurations
using :mc-cmd:`mc replicate export`, use :mc-cmd:`mc replicate import` to
reapply the bucket replication configuration.

See :ref:`minio-bucket-replication-serverside-twoway` or 
:ref:`minio-bucket-replication-serverside-multi` for a detailed procedure
on configuring active-active replication.

Once both deployments are fully resynchronized *and* replication rules are
working as expected, you can configure load balancers or proxies to begin
routing traffic normally.
