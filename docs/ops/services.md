# GRACC Services

These are the services the comprise the GRACC system, and details on
where and how they are currently deployed on GRACE. 

* Items marked with !! should be redesigned for production.
* Items marked with ?? should be evaluated for removal.
  (note: FIFE services will be moved to another Elasticsearch cluster "soon")


## External

### RabbitMQ

* Running at the GOC
* ITB: event-itb.grid.iu.edu:5672, :5671 (TLS), :15671 (HTTPS admin)
* Production: event.grid.iu.edu:5672, :5671 (TLS), :15671 (HTTPS admin)


## Head Node (`gratiav2-1`)

Openstack limits external access to ssh (:22) and http (:80 and :443).

### Elasticsearch (ingest node)

* Installed from official elastic repository
* Config in /etc/elasticsearch
* Running under systemd: `elasticsearch.service`
* Listening at:
    * http://0.0.0.0:9200 and :9300
    * https://gracc.opensciencegrid.org/e/
    * ?? https://fifemon-es.fnal.gov
* External access requires certificate from GRACC private CA
    * Certs and keys in `/etc/nginx/certs`
    * Run `make_cert COMMON_NAME` to generate and sign a new certificate
* nginx allows very limited read-only access through public endpoints


### Elasticsearch (query node)

* !! Manual copy of ingest node (`s/elasticsearch/elasticsearch-ro/`)
* Config in `/etc/elasticsearch-ro`
* Running under systemd: `elasticsearch-ro.service`
* Listening at:
  * http://localhost:9201
  * https://gracc.opensciencegrid.org/q/
* Includes `readonlyrest` plugin to allow limited unauthenticated read-only access 
  from Kibana, Grafana, and to public queries. Configured in `elasticsearch.yml`
  https://readonlyrest.com


### GRACC Collectors and Stash agents

GRACC Collectors (`gracc-collector`) accept records in formats provided by Gratia 
probes and collectors, transform them into GRACC JSON documents, and send them to 
the appropriate RabbitMQ exchange, which by convention is `gracc.$STREAM.raw`.

Raw Stash agents (`gracc-stash-raw`) create a queue in RabbitMQ that is bound the the appropriate
raw exchange, do minor manipulations to the documents (add checksum, calculate derived fields),
and index the documents into elasticsearch (`gracc.$STREAM.raw$N-$YYYY.$MM`), where `$N` is a 
monotonic schema version number. !! A cron job, currently running under kretzke, generates aliases 
in Elasticsearch each month from `gracc.$STREAM.raw$N-$YYYY.$MM` to `gracc.$STREAM.raw-$YYYY.$MM`.

Summary Stash agents (`gracc-stash-summary`) create a queue in RabbitMQ that is bound the the appropriate
summary exchange, do minor manipulations to the documents (add checksum, calculate derived fields),
and index the documents into elasticsearch (`gracc.$STREAM.summary$N-$YYYY`), where `$N` is a 
monotonic schema version number. !! Aliases are manually cretated in Elasticsearch from 
`gracc.$STREAM.summary$N-$YYYY` to `gracc.$STREAM.summary` (note just *one* alias for all years).

* Running under docker, via docker-compose
* Config in `/etc/gracc/docker/$STREAM`
* Within each directory is:
    * `docker-compose.yml` - defines services for the stream
    * `.env` - common environment variables for the stream's services

#### Streams
* `gracc.osg` - "main" job stream
    * Collector listening at:
        * http://localhost:8180
        * http://gracc.opensciencegrid.org/gracc/osg
* `gracc.osg-itb` - ITB test stream
    * Collector listening at:
        * http://localhost:8181
        * http://gracc.opensciencegrid.org/gracc/osg-itb
* `gracc.osg-transfer` - transfers stream
    * Collector listening at:
        * http://localhost:8182
        * http://gracc.opensciencegrid.org/gracc/osg-transfer


### GRACC Request Agent

The Request agent (`gracc-request`) listens on a RabbitMQ exchange (`gracc.$STREAM.requests`) 
for requests for "replay" or "summarization" of raw records for a given time period. The request 
includes a RabbitMQ exhchange the records should be sent to.

* !! Installed from copr (https://copr.fedorainfracloud.org/coprs/djw8605/GRACC/)
* Running under systemd: `graccreq.service`
* Config in `/etc/graccreq/config.d/gracc-request.toml`


### GRACC Summary Agent

The Summary agent (`gracc-summary`) periodically requests summarizations, and has a CLI to manually
summarize periods.


* !! Installed from copr (https://copr.fedorainfracloud.org/coprs/djw8605/GRACC/)
* Running under systemd: `graccsumperiodic.service` and `graccsumperiodic.timer`
* Config in `/etc/graccsum/config.d/gracc-summary.toml`

#### APEL reporting
* Source at [OSG docker](https://hub.docker.com/r/opensciencegrid/gracc-apel/) 
    * `docker {pull, run} opensciencegrid/gracc-apel`
* Running under systemd: `gracc-apel.service` and `gracc-apel.timer`
* Registered service certificate with [APEL admins](apel-admins@stfc.ac.uk) needed 
    * in docker we use:
    ```
        $ openssl x509 -in /etc/grid-security/apel/apelcert.pem -noout -subject
        subject= /DC=org/DC=opensciencegrid/O=Open Science Grid/OU=Services/CN=apel/hcc-grace.unl.edu
    ```
    * SELinux prevents inheritance of certificates, so this must be set on docker master node: `chcon -Rt svirt_sandbox_file_t /etc/grid-security/apel/`

##### Configuring GRACC-APEL in systemd 
* `gracc-apel.service` configure, enable, start, test
```
    $ cat /lib/systemd/system/gracc-apel.service 
      [Unit]
      Description=GRACC APEL reporting Docker container

      [Service]
      Type=oneshot
      ExecStart=/bin/docker run -v /etc/grid-security/apel/apelcert.pem:/etc/grid-security/apel/apelcert.pem -v /etc/grid-security/apel/apelkey.pem:/etc/grid-security/apel/apelkey.pem opensciencegrid/gracc-apel

    $ systemctl enabled gracc-apel.service
    $ systemctl start gracc-apel.service
    $ systemctl status gracc-apel.service
```

To run periodically:
* `gracc-apel.timer` configure, enable, start
```
    $ cat /lib/systemd/system/gracc-apel.timer 
    [Unit]
    Description=Run GRACC-to-APEL reporting script

    [Timer]
    # Explicitly declare service that this timer is responsible for
    Unit=gracc-apel.service
    # Runs 'gracc-apel' relative to when the *timer-unit* has been activated
    OnActiveSec=1hour
    # Runs 'gracc-apel' relative to when *service-unit* was last deactivated
    OnUnitInactiveSec=1hour

    # Randomize runtime by a small amount each run.
    RandomizedDelaySec=2min

    [Install]
    WantedBy=timers.target
    $ systemctl enabled gracc-apel.timer
    $ systemctl start gracc-apel.timer
    $ systemctl status gracc-apel.timer
```

    * To check if `gracc-timer.timer` runs:
```
    $ systemctl list-timers *apel*
    NEXT                         LEFT       LAST                         PASSED    UNIT             ACTIVATES
    Mon 2017-05-22 17:48:37 CDT  21min left Mon 2017-05-22 16:46:40 CDT  40min ago gracc-apel.timer gracc-apel.service
```

### Grafana

* Installed from official Grafana yum repo
* Running under systemd: `grafana-server.service`
* Config in `/etc/grafana`
* Data in `/var/lib/grafana`
* Listening at:
  * http://localhost:3000
  * https://gracc.opensciencegrid.org/


### Kibana

* Installed from official Elasticsearch yum repo
* Running under systemd: `kibana.service`
* Config in /etc/kibana
* Listening at:
  * http://localhost:5601
  * https://gracc.opensciencegrid.org/kibana/
* `readonlyrest` Elasticsearch plugin controls access, write actions require authentication
  with basic auth, configured in `/etc/elasticsearch-ro/elasticsearch.yml`.


### Prometheus

Prometheus is used for monitoring the nodes and services.

* !! Installed in `/opt/prometheus`
* Config in `/etc/prometheus`
* Data in `/data/prometheus`
* Running under systemd: `prometheus.service`
* Listening at:
  * http://localhost:9090
  * https://gracc.opensciencegrid.org/prometheus 
* !! external access requires basic auth, `/etc/nginx/conf.d/kibana.htpasswd`
* limited unauthenticated access to query endpoint

### Prometheus Exporters

Prometheus exporters collect data from services and present them for collection by Prometheus.

#### RabbitMQ Exporter
* !! manually run under docker (to be moved to docker-compose)
* configuration via env var
    * RABBIT_URL=https://event-itb.grid.iu.edu:15671
    * RABBIT_USER=$USER
    * RABBIT_PASSWORD=$PASSWORD
* Listening at http://localhost:9111

#### Node Exporter
* Installed at `/opt/prometheus`
* Run with systemd: `prometheus-node-exporter.service`
* Listening at http://0.0.0.0:9100

#### Graphite Exporter

The Graphite exporter accepts time-series data in graphite format and exposes it for 
collection by prometheus.

* Running under docker
* Listening at:
     * http://0.0.0.0:9100 (publish to prometheus)
     * https://0.0.0.0:2003 (collect from graphite)

### Nginx

All public services are proxied through nginx.

* Installed from EPEL
* Primary config in `/etc/nginx/conf.d/default.conf`
* Running under systemd: `nginx.service`
* Listening at:
  * http://0.0.0.0:80
  * http://0.0.0.0:443
  * http://gracc.opensciencegrid.org
  * https://gracc.opensciencegrid.org


### Docker

* Running under systemd: `docker.service`
* Extra config in `/etc/systemd/system/docker.service.d`
* Container logs sent to journald; e.g. `sudo journalctl CONTAINER_NAME=graccosg_gracc-collector_1`

#### Portainer

Portainer is a web-based utility for monitoring and managing containers.

* running under docker
* listening on http://0.0.0.0:9000
* !! not proxied through nginx, need to tunnel, e.g. `ssh -L 9000:localhost:9000 gracc.opensciencegrid.org`



## Data Nodes (`gratiav2-2` though `-5`)

Ansible is set up on the head node to perform operations across all data 
nodes, e.g. `sudo ansible elasticsearch -a 'uptime'`. Configuration in
`/etc/ansible`.

### Elasticsearch

* Installed from official elastic repository
* Config in `/etc/elasticsearch`
* Data in  `/data/elasticsearch` and `/data2/elasticsearch`
* Listening at http://0.0.0.0:9200 and :9300

### Prometheus Exporters

#### Node Exporter
* Installed at `/opt/prometheus`
* Run with systemd: `prometheus-elasticsearch-exporter.service`
* Listening at http://0.0.0.0:9100

#### Elasticsearch Exporter
* Installed at `/opt/prometheus`
* Run with systemd: `prometheus-node-exporter.service`
* Listening at http://0.0.0.0:9108
