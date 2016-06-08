# Message Queues

Message queues used in GRACC

---

In AMQP, there is a difference between a _queue_ and an _exchange_.  Messages delivered on a _queue_ are read by a single subscriber; messages delivered on an _exchange_ will be delivered to all subscribers (implying they may be buffered for some time at the broker if a given client goes missing).

We would like collectors to serve multiple databases (hence the use of an _exchange_) while queues are used for messages sent to a database agent.

Well known message queues and exchanges used:

* `/gracc.<collector>.raw` - An exchange which listens to raw records to insert into the collector.  This is the interface that probes would send raw records.
* `/grace.<db>.summary` - A queue that listens for summary records to insert into a specific `<db>`.  This is used to replicate summary records from other collectors or db's.
* `/grace.<db>.raw` - Raw record queue for a database instance.
* `/grace.<db>.requests` - The [Ad Agent](agent-arch.md) listens to this queue for requests for raw and summary replications.

Here, `<db>` is the instance name of a given database install while `<collector>` is the instance name of an existing Gratia collector.

There are currently three defined message schemas in GRACC: raw records, summary records, and replay requests:

Raw Records
-----------

These are JSON-formatted documents; the key-value pairs are derived from the OGF *UsageRecord* format.  For ease of compatibility with the prior Gratia system, we include an `njobs` attribute if a given record represents more than one job.

The [Raw Records](raw-records.md) page has more details and the mapping from XML UsageRecord.

```
{
    "RecordId": "osg-gw-7.t2.ucsd.edu:35741.2",
    "CreateTime": "2016-05-27T22:46:46Z",
    "GlobalJobId": "condor.osg-gw-7.t2.ucsd.edu#185777.0#1464388242",
    "LocalJobId": "185777",
    "LocalUserId": "cmsuser",
    "GlobalUsername": "cmsuser@t2.ucsd.edu",
    "DN": "/DC=ch/DC=cern/OU=Organic Units/OU=Users/CN=cmsuser/CN=1234567/CN=CMS User",
    "VOName": "/cms/Role=production/Capability=NULL",
    "ReportableVOName": "cms",
    "JobName": "osg-gw-7.t2.ucsd.edu#185777.0#1464388242",
    "MachineName": "osg-gw-7.t2.ucsd.edu",
    "SubmitHost": "osg-gw-7.t2.ucsd.edu",
    "Status": "0",
    "Status_description": "Condor Exit Status",
    "WallDuration": 617,
    "CpuDuration": 18,
    "CpuDuration_system": 18,
    "CpuDuration_user": 0,
    "EndTime": "2016-05-27T22:44:08Z",
    "StartTime": "2016-05-27T22:33:51Z",
    "Host": "cabinet-1-1-1.t2.ucsd.edu",
    "NodeCount": "1",
    "NodeCount_metric": "max",
    "Processors": "1",
    "Processors_metric": "max",
    "ResourceType": "Batch",
    "ProbeName": "condor:osg-gw-7.t2.ucsd.edu",
    "SiteName": "UCSDT2-D",
    "Grid": "OSG",
    "Njobs": "1",
}
```

!!! note
    We consider these to be "base" keys: additional ones may be given (for example, if the record is derived from a HTCondor ClassAd).


Summary Records
---------------

The summary record represents a grouping of multiple similar raw records.  In GRACC, we often group jobs run on the same date, by the same user, on the same resource.
```
TODO: copy JSON document here
```

Replay Requests
---------------

The replay request indicates that a remote listener agent attached to an ElasticSearch database should load and re-send some amount of data.

*Keys*:

* `from` and `to`: An ISO 8601 formatted date & time string that determines the time range beginning and ending, respectively, of the data to be sent.
* `kind`: What type of records should be resent (valid values are curently `raw` or `summary`).
* `destination`: An exchange on the same broker where records should be sent.  Should be a string value.
* `routing_key`: A routing key to be used when sending the data
* `control` and `control_key`: (optional) Control channel that will be notified when the data stream starts and ends.  Further, it will receive any errors that may occur during the replay.
* `filter`: (not implemented) A ElasticSearch-formatted query filter (JSON value).  Only records matching this filter should be sent.

Example
```json
{
  "from": "2016-05-10T00:00:00",
  "to": "2016-05-11T00:00:00",
  "kind": "raw",
  "destination": "grace.osg.raw",
  "control": "control-exchange",
  "control_key": "control_routing_key",
  "filter": {
    "query": {
      "query_string": {
        "query": "vo=cms"
      }
    }
  }
}
```

