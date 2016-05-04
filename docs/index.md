# GRACC

GRid ACcounting Collector

(pronounced "grok")

---

GRACC is a collection of components for implementing resource usage accounting.

### Why GRACC?

GRACC is meant as a replacement of the Gratia accounting system; the engineering focus is on smaller, independent components rather than a monolithic collector architecture.  The hope is that, by breaking the functionality into a series of smaller components, future architectural changes (such as migration to a new database) can be done without rewriting the entire infrastructure.  For example, forwarding information to a separate accounting database becomes much simpler in this infrastructure.

Repositories of interest:

* [GRACC Collector](https://github.com/opensciencegrid/gracc-collector).  An agent which runs on an existing Gratia collector that forwards raw usage records to GRACC.
* [GRACC Monitoring Emails](https://github.com/opensciencegrid/gracc-email).  Simple daily emails overview GRACC activity.
* [GRACC Request Daemon](https://github.com/opensciencegrid/gracc-request). Agents that listen for replay requests from GRACC.

