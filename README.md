orientdb-docker
===============

A dockerfile for creating an orientdb image with :

  - explicit orientdb version (1.7.8) for image cache stability
  - init by supervisord
  - config, databases and backup folders expected to be mounted as volumes

And lots of information from my orientdb+docker explorations. Read on!


Building the image on your own
==============================

1. Checkout this project to a local folder cding to it

2. Build the image:
`docker built -t <your_dockerhub_user>/orientdb-1.7.8 .`

3. Push it to your DockerHub repository (it will ask for your login credentials):
`docker push <your_dockerhub_user>/orientdb-1.7.8`

All examples below are using my own image nesrait/orientdb-1.7.8. If you build your own image please find/replace "nesrait" with your Docker Hub user.


Running orientdb
================

To run the image, run:

`docker run --name orientdb -d -v <config_path>:/opt/orientdb/config -v <databases_path>:/opt/orientdb/databases -v <backup_path>:/opt/orientdb/backup -p 2424 -p 2480 nesrait/orientdb-1.7.8`

The docker image contains a unconfigured orientdb installation and for running it you need to provide your own config folder from which OrientDB will read its startup settings.

The same applies for the databases folder which if local to the running container would go away as soon as it died/you killed it.

The backup folder only needs to be mapped if you activate that setting on your OrientDB configuration file.


Persistent distributed storage using BTSync
===========================================

If you're not running OrientDB in a distributed configuration you need to take special care to backup your database (in case your host goes down).

Below is a simple, yet hackish, way to do this: using BTSync data containers to propagate the OrientDB config, LIVE databases and backup folders to remote location(s).
Note: don't trust the remote copy of the LIVE database folder unless the server is down and it has correctly flushed changes to disk. 

1. Create BTSync shared folders on any remote location for the various folder you want to replicate
  1.1 - config: orientdb configuration inside the config folder
  1.2 - databases: the LIVE databases folder
  1.3 - backup: the place where OrientDB will store the zipped backups (if you activate the backup in the configuration file)

2. Take note of the BTSync folder secrets <CONFIG_FOLDER_SECRET>, <DATABASES_FOLDER_SECRET>, <BACKUP_FOLDER_SECRET>

3. Launch BTSync data containers for each of the synched folder you created giving them proper names:
`docker run -d --name orientdb_config -v /opt/orientdb/config nesrait/btsync /opt/orientdb/config <CONFIG_FOLDER_SECRET>`
`docker run -d --name orientdb_databases -v /opt/orientdb/databases nesrait/btsync /opt/orientdb/databases <DATABASES_FOLDER_SECRET>`
`docker run -d --name orientdb_backup -v /opt/orientdb/backup nesrait/btsync /opt/orientdb/backup <BACKUP_FOLDER_SECRET>`

3. Wait until all files have magically appeared inside your BTSync data volumes:
`docker run --rm -i -t --volumes-from orientdb_config --volumes-from orientdb_databases --volumes-from orientdb_backup ubuntu du -h /opt/orientdb/config /opt/orientdb/databases /opt/orientdb/backup`

4. Finally you're ready to start your OrientDB server
`docker run --name orientdb -d \
            --volumes-from orientdb_config \
            --volumes-from orientdb_databases \
            --volumes-from orientdb_backup \
            -p 2424 -p 2480 \
            nesrait/orientdb-1.7.8`


OrientDB distributed
====================

If you're running OrientDB distributed* you won't have the problem of losing the contents of your databases folder since they are already replicated to the other OrientDB nodes. From the setup above simply leave out the "--volumes-from orientdb_databases" argument and OrientDB will use the container storage to hold your databases' files.

*note: some extra work might be needed to correctly setup hazelcast running inside docker containers (see https://groups.google.com/forum/#!topic/vertx/MvKcz_aTaWM).


Ad-hoc backups
==============

With orientdb 1.7.8 we can now create ad-hoc backups by taking advantage of the new backup.sh script:
https://github.com/orientechnologies/orientdb/wiki/Backup-and-Restore#backup-database

  - Using the orientdb_backup data container that was created above:
    `docker run -i -t --volumes-from orientdb_config --volumes-from orientdb_backup nesrait/orientdb-1.7.8 ./backup.sh <dburl> <user> <password> /opt/orientdb/backup/<backup_file> [<type>]`

  - Or using a host folder:
    `docker run -i -t --volumes-from orientdb_config -v <host_backup_path>:/backup nesrait/orientdb-1.7.8 ./backup.sh <dburl> <user> <password> /backup/<backup_file> [<type>]`

Either way, when the backup completes you will have the backup file located outside of the OrientDB container and read for safekeeping.

Note: I haven't tried the non-blocking backup (type=lvm) yet but I read about some issues with a docker LVM dependency:
https://groups.google.com/forum/#!topic/docker-user/n4Xtvsb4RAw


Running the orientdb console
============================

`docker run --rm -it \
            --volumes-from orientdb_config \
            --volumes-from orientdb_databases \
            --volumes-from orientdb_backup \
            nesrait/orientdb-1.7.8 \
            /opt/orientdb/bin/console.sh`