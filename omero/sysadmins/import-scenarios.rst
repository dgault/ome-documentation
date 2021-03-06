Advanced import scenarios
=========================

Increasingly users of OMERO are needing to go beyond the traditional "upload
via a GUI"-style import model to more powerful methods.

There is a set of requirements for getting data into OMERO that is common
to many institutions. Some of the requirements may be mutually exclusive.

* Users need to get data off microscopes quickly. This likely includes
  not waiting for import to complete. Users will often move data immediately,
  or even save remotely during acquisition.

* Users would like direct access to the binary repository file-system
  to read original files for analysis.

* Users would like to view and begin working with images as soon as
  possible after acquisition.

Below we explain which options are available to you, and why there is a
trade-off between the above requirements.

Import overview
---------------

The “OMERO binary repository” (or repo) is the directory belonging to the
OMERO user where files are imported:

* The :doc:`ManagedRepository
  </developers/Server/FS>` directory inside of
  the repo is where files go during import into OMERO. Each user receives a
  top-level directory inside of “ManagedRepository” which fills with
  timestamped directories as imports accrue.

* Depending on the permissions of this directory, users may or may not be
  able to see their imported files. Managing the permissions is the
  responsibility of the system administrator.

In “normal import”, files are copied to the OMERO binary repo via the API
and so can work remotely or locally. In :doc:`“in-place import”
<in-place-import>`, files are “linked” into place.

.. warning:: In-place import is a new, powerful feature - it is critical that
    you read and understand the :doc:`documentation <in-place-import>` before
    you consider using it.

Traditional import
------------------

Manual import (GUI)
^^^^^^^^^^^^^^^^^^^

This is the standard workflow and the one currently used
at the University of Dundee. Users dump data to a shared
file-system from the acquisition system, and then use the
OMERO.insight client from the lab to import.

Advantages
""""""""""

* Users can validate that import worked.

* Failed imports can be repeated and/or reported to QA etc.

* Users do not have to wait for import to be scheduled.

* Import destination is known: Project/Dataset etc.

Disadvantages
"""""""""""""

* Imports can be slow due to the data transfer from file-system to OMERO via
  the client.

* Users must remember to delete data from the shared file-system to avoid data
  duplication.

* Users cannot access the OMERO binary repo directly and must download
  original data via clients for local analysis.

Manual import (CLI)
^^^^^^^^^^^^^^^^^^^

Another typical workflow is that rather than using the GUI, users perform the
same procedure as under "Manual import" but with the
:doc:`command-line (CLI) importer </users/cli/import>`.

Advantages
""""""""""

* With a CLI workflow, it may be easier for users to connect remotely
  to kick off an import and to leave it running in the background for a long
  period of time.

Disadvantages
"""""""""""""

All the same disadvantages apply as under "Manual import (GUI)".

Cronjob import (manual delete)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

For importing via cron, users still dump their data to a shared file-system
from the acquisition system. They must have permissions to write to “their”
directory which is mapped to a user in OMERO.

A cronjob starts a CLI import, possibly at night. The cronjob could be given
admin rights in OMERO to perform an "Import As" for a particular user.

Disadvantages
"""""""""""""

* If a normal import is used, the cronjob would have to manually delete
  imported files from their original location to avoid duplication.

* Users cannot work with their data in OMERO until some time after
  acquisition.

* Failed imports are logged within the managed repository but not yet
  notified.
  Logs would probably need to be accessed via a sysadmin/cli. The cronjob
  could capture stdout and stderr and check for failures.

DropBox import (manual delete)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Similar to the cronjob scenario, DropBox importing requires that users drop
their data in "their" directory which has special permissions for writing.
The DropBox service monitors those directories for modifications and imports
the files on a first-come-first-serve basis.

* :doc:`/sysadmins/dropbox`

Advantages
""""""""""

* Users should see their data in OMERO quickly.

Disadvantages
"""""""""""""

* There is a limitation on the rate of new files in monitored locations.

* There is also a limitation on which file systems can be used. A networked
  file share **cannot** be monitored by DropBox.

* Users must manually delete imported files from their DropBox directory to
  avoid duplication.

* Failed imports are logged within the managed repository but not yet
  notified.
  Logs would probably need to be accessed via a sysadmin or through the CLI
  and searched by the user and file name.

.. _upload_dropbox_auto:

DropBox import (automatic delete)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

One option is to have files removed from DropBox automatically after a
successful import. This is achieved by performing an "upload" import
from the DropBox directory to the ManagedRepository then deleting the
data from DropBox **if and only if** the import was successful. For
failed imports, files will remain in the DropBox directories until
someone manually deletes them.

Advantages
""""""""""

* For all successful imports, files will be automatically removed
  from the DropBox directories thus reducing duplication.

In-place import
---------------

The following sections outline :ref:`in-place <inplace_import>` based
scenarios to help you judge if the functionality may be useful for you.

Common advantages
^^^^^^^^^^^^^^^^^

* All in-place import scenarios provide non-copying benefit. Data that is
  too large to exist in multiple places, or which is accessed too frequently
  in its original form to be renamed, remains where it was originally
  acquired.

Common disadvantages
^^^^^^^^^^^^^^^^^^^^

* Like the DropBox import scenario above, all in-place imports require the
  user to have access to the user-based directories under the
  ManagedRepository. See :ref:`limitations <limitations>` for more details.

* Similarly, all the following scenarios carry the same burden of securing
  the data externally to OMERO. This is the primary difference between a
  normal import and an in-place import: **backing up OMERO is no longer
  sufficient to prevent data loss. The original location must also be
  secured!** This means that users must not move or alter data once imported.

In-place manual import (CLI)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The in-place version of a CLI manual import is quite similar to the normal
CLI import, with the primary difference being that the data is not transferred
from the shared file-system where the data is initially stored after
acquisition, but instead is just "linked" into place.

Advantages
""""""""""

* Local filesystem in-place import is faster than traditional importing, due
  to the lack of a data transfer.

Disadvantages
"""""""""""""

* Requires proper security setup as explained above.

In-place Cronjob import
^^^^^^^^^^^^^^^^^^^^^^^

Assuming all the restrictions are met, the cronjob-based workflow above
can carry out an in-place import by adding the in-place transfer flag. The
advantages and disadvantages are as above.

.. _inplace_dropbox_manual:

In-place DropBox import (manual delete)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Just as with the in-place cronjob import, using in-place import for DropBox
is as straight-forward as passing the in-place flag. The common advantages
and disadvantages of in-place import apply.

.. _inplace_dropbox_auto:

In-place DropBox import (automatic delete)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

An option that also exists in the in-place scenario is to have
files removed from DropBox automatically after a successful import.
This is achieved by first performing a "hardlink in-place import" from
the DropBox directory to the ManagedRepository and then by deleting
the data from DropBox **if and only if** the import was successful. For
failed imports, files will remain in the DropBox directories until someone
manually deletes them.

Advantages
""""""""""

* For all successful imports, files will be automatically removed
  from the DropBox directories.

Disadvantages
"""""""""""""

* This option is only available if the filesystem which DropBox watches is
  the same as the file system which the ManagedRepository lives on. This
  prevents the use of network file systems and similar remote shares.

.. seealso:: 

    :doc:`/sysadmins/in-place-import`

    :doc:`/sysadmins/dropbox`

    :doc:`/users/cli/import`

    :doc:`/users/cli/import-target`

