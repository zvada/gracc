# Troubleshooting of GRACC services

## Helpful dashboards
* [GRACC Service Status](https://gracc.opensciencegrid.org/dashboard/db/gracc-service-status?refresh=1m&orgId=1)
* [GRACC Collector Stats](https://gracc.opensciencegrid.org/dashboard/db/gracc-collector-stats?refresh=1m&orgId=1)
* [RabbitMQ queues](https://gracc.opensciencegrid.org/dashboard/db/rabbitmq-queues?refresh=1m&orgId=1)
* [Probe Record Rate](https://gracc.opensciencegrid.org/dashboard/db/probe-record-rate?orgId=1&var-Probe=slurm:grid1.oscer.ou.edu) - example for given CE
    * in addition, check on [Kibana ProbeName records](<https://gracc.opensciencegrid.org/kibana/app/kibana#/discover?_g=(refreshInterval:(%27$$hashKey%27:%27object:1968%27,display:%271%20minute%27,pause:!f,section:2,value:60000),time:(from:now-7d,mode:quick,to:now))&_a=(columns:!(_source),index:%27gracc.osg.raw-*%27,interval:auto,query:(query_string:(analyze_wildcard:!f,query:%27ProbeName:%22slurm:grid1.oscer.ou.edu%22%27)),sort:!(EndTime,desc))>)
* [OSG Connect Summary - UChicago](https://gracc.opensciencegrid.org/dashboard/db/osg-connect-summary-uchicago-ci?from=now-30d&to=now&orgId=1)
* [Site Transfer Summary](https://gracc.opensciencegrid.org/dashboard/db/site-transfer-summary?orgId=1&from=1454284800000&to=1485907200000&var-interval=$__auto_interval&var-site=All)
* [Institutions contributing to the OSG by name](https://gracc.opensciencegrid.org/dashboard/db/payload-jobs-summary?panelId=15&fullscreen&orgId=1)

## Issues

Selection of issues being investigated and actions taken in order to resolve them.

### Logstash

__Symptom__: 

* high usage of gracc archiver memory (e.g. ~12GB)
* logstash seems to be backed up and not responding
* RabbitMQ has high volume of queued messages (e.g. ~100k)

__Action__: 

* `systemctl restart elasticsearch.service`
* added to [check_mk](https://hcc-mon.unl.edu/red/check_mk/index.py?start_url=%2Fred%2Fcheck_mk%2Fview.py%3Fview_name%3Dhost%26host%3Dgracc.opensciencegrid.org)systemd monitoring for elasticsearch and elasticsearch-ro  
* for continuous high rate disconnections in RabbitMQ contact [Marina Krenz](mailto:mvkrenz@iu.edu)

### GRACC-APEL

#### Update missing records

* It may happen site has problem with sending accouting data to GRACC in particular month so when fixed they ask us correct accoutning in APEL report. In such case do:
    1) From `hcc-grace-itb.unl.edu` run manually `$ cd /root/gracc-apel/; ./apel_report YYYY MM`

    2) Move file `$ mv /root/gracc-apel/MM_YYYY.apel /var/spool/apel/outgoing/12345678/1234567890abcd`

    3) Send off `$ ssmsend`
* WLCG reports generated from APEL accouting are stored [here](https://espace.cern.ch/WLCG-document-repository/Accounting/Forms/AllItems.aspx).
