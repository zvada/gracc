Backup Configuration
====================

Backup Sources
--------------

GRACC backup sources come from listening and duplicating all raw records sent to the system through the RabbitMQ system.  The [GRACC Archiver](https://github.com/opensciencegrid/gracc-archive) listens to the raw RabbitMQ exchanges.  It listens to both the `gracc.osg.raw` and the `gracc.osg-transfer.raw` exchanges.

The archive agent stores the records into a tar.gz file located in `/var/lib/graccarchive/sandbox`.  On new days (or agent crashes) the tar.gz files are atomically copied to `/var/lib/graccarchive/output`.  The transfer archiver similarily stores files in `/var/lib/graccarchive/sandbox-transfer` and `/var/lib/graccarchive/output-transfer`.

The archive agent is configured in `/etc/graccarchive/conf`.  It uses systemd template units to run both the raw jobs and raw transfer archivers at the same time.


Backup Location
---------------

The backups are copied to FNAL by the [gracc-backup](https://github.com/opensciencegrid/gracc-backup) tool.  This uses SystemD timer and service files to periodically copy the output `.tar.gz` files to FNAL.  The final destination (gsiftp) of the files is configured in the *.service files with the tool


Restore Operation
-----------------

The restore operation uses the `unarchive` tool distributed with the GRACC Archiver agent.  The workflow of a restore is:

1. Copy the backup file from the backup location.
2. Run the `unarchive` tool from the GRACC Archiver on the compressed .tar.gz file, with command line arguments for the RabbitMQ parameters.

