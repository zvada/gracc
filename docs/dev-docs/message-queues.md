# Message Queues

Message queues used in GRACC

---

Well known message queues used:

* `/gracc.<collector>.raw` - Listens to raw records to insert into the collector.  This is the interface that probes would send raw records.
* `/gracc.<db>.summary` - Listens for summary records to insert into a specific `<db>`.  This is used to replicate summary records from other collectors or db's.
* `/gracc.<db>.raw` - 
* `/gracc.<db>.requests` - The [Ad Agent](agent-arch.md) listens to this queue for requests for raw and summary replications.