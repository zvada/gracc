
Installing GRACE
================

The GRACE database service consists of:

   - ElasticSearch as a datastore.
   - `grace-raw-listener`: Listens to the raw records from the collector
   - `grace-summary-listener`: Listens for and requests summary records.
   - `grace-request-listener`: Listens for replay and summarization requests.

See the [agent architecture docs](agent-arch.md) for more information.

Additionally, for monitoring and visualization, one commonly installs
the following external components.

   - InfluxDB (primarily for monitoring ES performance)
   - Grafana (visualization)
   - Kibana (visualization)

Dependencies
------------

This document assumes a RHEL7 host with ElasticSearch pre-installed and functioning.

We also assume that the OSG repos and `yum` priorities have been [appropriately configured](http://opensciencegrid.github.io/docs/Common/yum/).

Installation
------------

The relevant components can be pulled in via a meta-RPM:
```
yum install --enablerepo=osg-development osg-grace
```

Configuration
-------------

Configuration files are kept in `/etc/gracc/config.d` and `/usr/share/gracc/config.d`; files in this directory are read in lexigraphical order.  The file format is [TOML](https://github.com/toml-lang/toml).

You will likely need to override at least the AMQP connection paramaters (username and password).  Do not edit the default file in `/usr/share/gracc/config.d`: these will be overwritten on upgrade.  Instead, start by editing the samples in `/etc`.

Most strings will expand the text %I to the "instance name".  So, for the `osg` instance, the following,

```
[AMQP]
exchange = "gracc.%I.raw"
queue = "grace.%I.raw"
```

is equivalent to

```
[AMQP]
exchange = "gracc.osg.raw"
queue = "grace.osg.raw"
```

Further, we can have specific instance overrides.  Hence,

```
[AMQP]
exchange = "gracc.%I.raw"
queue = "grace.%I.raw"

[AMQP.osg]
exchange = "gracc.osg-test.raw"
```

is equivalent to:

```
[AMQP]
exchange = "gracc.osg-test.raw"
queue = "gracc.osg.raw"
```

### Running services

To configure GRACC to start at boot, you will need to do the following for the `osg` instance:

```
ln -sf /usr/lib/systemd/system/grace-raw-listener@.service /etc/systemd/system/multi-user.target.wants/grace-raw-listener@osg.service
ln -sf /usr/lib/systemd/system/grace-summary-listener@.service /etc/systemd/system/multi-user.target.wants/grace-summary-listener@osg.service
ln -sf /usr/lib/systemd/system/grace-request-listener@.service /etc/systemd/system/multi-user.target.wants/grace-request-listener@osg.service
systemctl daemon-reload
```

Multiple instance can be run by editing the instance name above.

Finally, these services can be started via the typical system management commands:

```
systemctl start grace-raw-listener@osg
systemctl start grace-summary-listener@osg
systemctl start grace-request-listener@osg
```

### Log files

By default, log files go into `/var/log/gracc`.

