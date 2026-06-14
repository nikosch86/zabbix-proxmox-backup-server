# Proxmox Backup Server by HTTP

## Overview

Template for monitoring Proxmox Backup Server (PBS) over its REST API using the Zabbix HTTP agent.

It collects the API service status, datastore usage (including a linear-regression estimate of when each datastore will be full), failed/erroneous tasks, per-backup-group snapshot freshness and the physical disk inventory with its SMART status.

PBS uses an API, the documentation can be found here: https://pbs.proxmox.com/docs/api-viewer/index.html

## Requirements

Zabbix version 7.2 and higher.

## Tested versions

This template has been tested on Proxmox Backup Server 3.1 .. 3.4.1

## Author

nikosch86

Source and issue tracker: https://github.com/nikosch86/zabbix-proxmox-backup-server

## Setup

Create a monitoring user and a corresponding API Token.

Set the following access levels for the User and the Token:

- Check: `["perm","/",["Audit"]]`

Use the resulting Token ID and Secret in the host macros.

### TLS certificate verification

The template verifies the PBS server's TLS certificate by default and will **not** connect to a server presenting an untrusted (e.g. self-signed) certificate. The HTTP agent items have *SSL verify peer* and *SSL verify host* enabled, and the `PBS: API service status` script item validates the certificate as well.

PBS ships with a self-signed certificate by default. To monitor such a server, add its certificate (or the issuing CA) to the trusted CA store of the Zabbix server — and of every Zabbix proxy that runs the checks — and point `SSLCALocation` at it, then restart the service.

Trusting the certificate at the server is the only option that also covers the API status check: that script item verifies independently and, because the embedded `HttpRequest` object exposes no SSL toggle, its verification cannot be disabled per item. Merely unchecking *SSL verify peer/host* on the HTTP agent items would leave the API status check failing.

### Macros used

| Name                           | Description                                                                                                                                              | Default                                |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------- |
| {$PBS.HOST}                    | <p>Hostname or IP address of the PBS server API endpoint. Must be set on the host.</p>                                                                  | `<SET PBS HOSTNAME>`                           |
| {$PBS.PORT}                    | <p>The API uses the HTTPS protocol and the server listens on port 8007 by default.</p>                                                                  | `8007`                                 |
| {$PBS.TOKEN.ID}                | <p>API tokens allow stateless access to most parts of the REST API by another system, software or API client.</p>                                       | `USER@REALM!TOKENID`                   |
| {$PBS.TOKEN.SECRET}            | <p>Secret key of the API token. Typed as a secret macro — set it per host; the value is not stored in the template export.</p>             | _(secret, set per host)_               |
| {$PBS.DATASTORE.AVAILABLE.MIN} | <p>Minimum available space in a datastore, in bytes (defaults to 10 GiB). Can be overridden per datastore using the `{#DATASTORE.NAME}` macro context.</p> | `10737418240`                          |
| {$PBS.TASKS.DAYS}              | <p>Age of tasks to consider when looking for failed tasks (days).</p>                                                                                   | `2`                                    |
| {$PBS.SNAPSHOT.AGE.WARN}       | <p>Last-snapshot age of a backup group above which a WARNING severity trigger fires. Default suits a daily backup schedule. Can be overridden per backup group using the `{#BACKUP.GROUP}` macro context.</p> | `30h`                                  |
| {$PBS.SNAPSHOT.AGE.HIGH}       | <p>Last-snapshot age of a backup group above which a HIGH severity trigger fires. Default suits a daily backup schedule. Can be overridden per backup group using the `{#BACKUP.GROUP}` macro context.</p>    | `50h`                                  |
| {$PBS.SNAPSHOT.INTERVAL}       | <p>Polling interval of the backup group list used for snapshot freshness. Listing groups scans every backup group of each datastore, so keep this interval moderate.</p> | `15m`                                  |
| {$PBS.SNAPSHOT.IGNORE.COMMENT} | <p>Regular expression matched against a backup group's comment; a match suppresses that group's snapshot-freshness triggers (the age item is kept, only the triggers are dropped). Use it for groups whose source VM/CT was deleted but whose backups are intentionally retained — see [Backup groups whose source was deleted](#backup-groups-whose-source-was-deleted).</p> | `(?i)no-monitor` |
| {$PBS.LLD.DISABLE.LOST}        | <p>Grace period after which a discovered datastore or backup-group item is disabled once its datastore/group no longer exists in PBS, instead of polling the gone resource and erroring for the full keep period. Must exceed the discovery interval; the default is ≈ 4 LLD poll cycles, which debounces a single incomplete API poll. Does not apply to disk discovery.</p> | `1h`  |
| {$PBS.LLD.KEEP.LOST}           | <p>Period after which a no-longer-present, already-disabled discovered datastore or backup-group item is deleted from the host. Must exceed `{$PBS.LLD.DISABLE.LOST}`. Does not apply to disk discovery.</p> | `14d` |

### Items

| Name                      | Description                       | Type       | Key and additional info                                                                                                                                                                                                                                              |
| ------------------------- | --------------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| PBS: API service status   | <p>Get API service status.</p>    | Script     | pbs.api.available<p>**Preprocessing**</p><ul><li><p>Discard unchanged with heartbeat: `12h`</p></li></ul><p>Value map: `HTTP response status code`</p>                                                                                                              |
| PBS: Get datastore status | <p>Get datastore status.</p>      | HTTP agent | pbs.datastore.status<p>**Preprocessing**</p><ul><li><p>Check for not supported value</p><p>⛔️Custom on fail: Set value to: `Error getting data`</p></li><li><p>JSONPath: `$.body.data`</p></li></ul>                                                                |
| PBS: Get disks            | <p>Get disks.</p>                 | HTTP agent | pbs.disks<p>**Preprocessing**</p><ul><li><p>Check for not supported value</p><p>⛔️Custom on fail: Set value to: `Error getting data`</p></li><li><p>JSONPath: `$.body.data`</p></li></ul>                                                                            |
| PBS: Get failed tasks     | <p>Get erroneuous tasks.</p>      | HTTP agent | pbs.tasks.error<p>**Preprocessing**</p><ul><li><p>Check for not supported value</p><p>⛔️Custom on fail: Set value to: `Error getting data`</p></li><li><p>JSONPath: `$.body.data`</p></li><li><p>JavaScript: filters tasks newer than `{$PBS.TASKS.DAYS}` days</p></li></ul> |
| PBS: Get backup groups    | <p>Backup groups and their latest snapshot time (`last-backup`), across all datastores, from the per-datastore groups endpoint. One entry per backup group is stored per poll, including the group `comment` so freshness alerting can be suppressed for intentionally-retained groups. Datastores that report an error or cannot be queried are skipped.</p> | Script | pbs.backupgroups<p>Update interval: `{$PBS.SNAPSHOT.INTERVAL}`</p> |
| PBS: Failed backup tasks  | <p>Failed backup tasks (worker type prefix `backup`) within the last {$PBS.TASKS.DAYS} days.</p> | Dependent item | pbs.tasks.error.worktype[backup]<p>**Preprocessing**</p><ul><li><p>JavaScript: keeps tasks whose `worker_type` starts with `backup`</p></li></ul> |
| PBS: Failed garbage collection tasks | <p>Failed garbage collection tasks (worker type prefix `garbage_collection`) within the last {$PBS.TASKS.DAYS} days.</p> | Dependent item | pbs.tasks.error.worktype[garbage_collection]<p>**Preprocessing**</p><ul><li><p>JavaScript: keeps tasks whose `worker_type` starts with `garbage_collection`</p></li></ul> |
| PBS: Failed prune tasks   | <p>Failed prune tasks (worker type prefix `prune`, covering manual and scheduled prune jobs) within the last {$PBS.TASKS.DAYS} days.</p> | Dependent item | pbs.tasks.error.worktype[prune]<p>**Preprocessing**</p><ul><li><p>JavaScript: keeps tasks whose `worker_type` starts with `prune`</p></li></ul> |
| PBS: Failed sync tasks    | <p>Failed sync tasks (worker type prefix `sync`, covering manual syncs and scheduled sync jobs) within the last {$PBS.TASKS.DAYS} days.</p> | Dependent item | pbs.tasks.error.worktype[sync]<p>**Preprocessing**</p><ul><li><p>JavaScript: keeps tasks whose `worker_type` starts with `sync`</p></li></ul> |
| PBS: Failed verify tasks  | <p>Failed verify tasks (worker type prefix `verif`, covering manual verification and scheduled verification jobs) within the last {$PBS.TASKS.DAYS} days.</p> | Dependent item | pbs.tasks.error.worktype[verify]<p>**Preprocessing**</p><ul><li><p>JavaScript: keeps tasks whose `worker_type` starts with `verif`</p></li></ul> |
| PBS: Failed other tasks   | <p>Failed tasks within the last {$PBS.TASKS.DAYS} days whose worker type is not covered by the per-worktype items.</p> | Dependent item | pbs.tasks.error.other<p>**Preprocessing**</p><ul><li><p>JavaScript: keeps tasks whose `worker_type` matches none of the per-worktype prefixes</p></li></ul> |

### Triggers

| Name                           | Description                                                                                | Expression                                                  | Severity | Dependencies and additional info |
| ------------------------------ | ----------------------------------------------------------------------------------------- | ---------------------------------------------------------- | -------- | -------------------------------- |
| PBS: API service not available | <p>The API service is not available. Check the network connection, the PBS service status and the API token authorization settings.</p> | `last(/Proxmox Backup Server by HTTP/pbs.api.available)<>200` | High     |                                  |
| PBS: Failed backup tasks found | <p>Backup tasks failed within the last {$PBS.TASKS.DAYS} days. A client backup did not complete.</p> | `last(/Proxmox Backup Server by HTTP/pbs.tasks.error.worktype[backup])<>"[]"` | High | <p>**Depends on:**</p><ul><li>PBS: API service not available</li></ul> |
| PBS: Failed garbage collection tasks found | <p>Garbage collection tasks failed within the last {$PBS.TASKS.DAYS} days. Unused chunks are not being reclaimed.</p> | `last(/Proxmox Backup Server by HTTP/pbs.tasks.error.worktype[garbage_collection])<>"[]"` | Average | <p>**Depends on:**</p><ul><li>PBS: API service not available</li></ul> |
| PBS: Failed prune tasks found  | <p>Prune tasks failed within the last {$PBS.TASKS.DAYS} days. Old snapshots are not being cleaned up.</p> | `last(/Proxmox Backup Server by HTTP/pbs.tasks.error.worktype[prune])<>"[]"` | Average | <p>**Depends on:**</p><ul><li>PBS: API service not available</li></ul> |
| PBS: Failed sync tasks found   | <p>Sync tasks failed within the last {$PBS.TASKS.DAYS} days. Off-site copies are not being replicated.</p> | `last(/Proxmox Backup Server by HTTP/pbs.tasks.error.worktype[sync])<>"[]"` | High | <p>**Depends on:**</p><ul><li>PBS: API service not available</li></ul> |
| PBS: Failed verify tasks found | <p>Verification tasks failed within the last {$PBS.TASKS.DAYS} days. Snapshot integrity is not confirmed.</p> | `last(/Proxmox Backup Server by HTTP/pbs.tasks.error.worktype[verify])<>"[]"` | Average | <p>**Depends on:**</p><ul><li>PBS: API service not available</li></ul> |
| PBS: Failed tasks found        | <p>Failed tasks occured within the last {$PBS.TASKS.DAYS} days. Catch-all for worker types not covered by the per-worktype triggers.</p> | `last(/Proxmox Backup Server by HTTP/pbs.tasks.error.other)<>"[]"`  | High     | <p>**Depends on:**</p><ul><li>PBS: API service not available</li></ul> |

### LLD rule Backup group discovery

| Name                       | Description                                                                                                                                                                                             | Type           | Key and additional info  |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------- | ------------------------ |
| PBS: Backup group discovery | <p>Discovers the backup groups (`backup-type/backup-id`, e.g. `host/myclient`) that have at least one snapshot in any datastore.</p> | Dependent item | pbs.backupgroup.discovery |

> **Caveat:** a client that has never produced a single snapshot is not discovered and therefore not monitored for freshness. This rule catches backups that *stopped*, not backups that were never set up.

### Item prototypes for Backup group discovery

| Name                                                                  | Description                                                                                                                                                                                                                                       | Type           | Key and additional info |
| --------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------- | ----------------------- |
| PBS: Backup group [{#DATASTORE.NAME}/{#BACKUP.GROUP}] Last Snapshot Age | <p>Seconds since the most recent snapshot of the backup group. For datastores filled by a sync job the snapshots keep the original backup-time of the source, so on the sync target this measures end-to-end freshness (client backup plus sync).</p> | Dependent item | pbs.backupgroup.lastsnapshot.age[{#DATASTORE.NAME},{#BACKUP.GROUP}]<p>**Preprocessing**</p><ul><li><p>JSONPath: selects the group's latest backup-time</p><p>⛔️Custom on fail: Discard value</p></li><li><p>JavaScript: seconds since that time, clamped to `0`</p></li></ul> |

### Trigger prototypes for Backup group discovery

| Name                                                                                       | Description                                                                                                                                              | Expression                                                                                                                                      | Severity | Dependencies and additional info |
| ------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- | -------- | -------------------------------- |
| PBS: Backup group [{#DATASTORE.NAME}/{#BACKUP.GROUP}] last snapshot older than high threshold    | <p>No new snapshot for an extended period; check the client and its backup schedule.</p>                                                                  | `last(/Proxmox Backup Server by HTTP/pbs.backupgroup.lastsnapshot.age[{#DATASTORE.NAME},{#BACKUP.GROUP}])>{$PBS.SNAPSHOT.AGE.HIGH:"{#BACKUP.GROUP}"}` | High     | <p>**Depends on:**</p><ul><li>PBS: API service not available</li></ul> |
| PBS: Backup group [{#DATASTORE.NAME}/{#BACKUP.GROUP}] last snapshot older than warning threshold | <p>A recent client backup run may have failed or been skipped.</p>                                                                                        | `last(/Proxmox Backup Server by HTTP/pbs.backupgroup.lastsnapshot.age[{#DATASTORE.NAME},{#BACKUP.GROUP}])>{$PBS.SNAPSHOT.AGE.WARN:"{#BACKUP.GROUP}"}` | Warning  | <p>**Depends on:**</p><ul><li>PBS: API service not available</li><li>PBS: Backup group [{#DATASTORE.NAME}/{#BACKUP.GROUP}] last snapshot older than high threshold</li></ul> |

### Backup groups whose source was deleted

When a VM or CT is deleted on the Proxmox VE side, its backups remain in PBS but no new snapshot will ever be added. The group's *Last Snapshot Age* keeps climbing, so the freshness triggers fire and — because the condition stays true forever — **never recover on their own**.

PBS cannot tell this apart from a genuinely broken backup job: from the groups endpoint, "source deleted" and "backups stopped" both look like "`last-backup` is getting old". There is no source-exists flag to key on, so the distinction has to come from you. Two ways, depending on whether you want it permanent:

- **Permanent — mark the group in PBS (recommended).** In PBS, open the datastore's *Content* view, edit the backup group's comment and include the marker `no-monitor` (configurable via `{$PBS.SNAPSHOT.IGNORE.COMMENT}`). On the next discovery run the LLD override sets both freshness trigger prototypes to *not discovered* for that group, so its freshness triggers are no longer (re)created. The *Last Snapshot Age* item is not targeted by the override and is kept, so you can still see how stale the group is. Remove the marker and the triggers come back — the single source of truth lives in PBS, not in Zabbix.
  - **If the triggers have not fired yet,** this is clean: no problem is ever raised for the group.
  - **If the group is *already* alerting,** the override does **not** delete the existing triggers right away and does **not** resolve the open problem on the next discovery run. The triggers become *lost resources*: this rule disables lost resources after `{$PBS.LLD.DISABLE.LOST}` (default `1h`) and deletes them after `{$PBS.LLD.KEEP.LOST}` (default `14d`). So for roughly the first hour the age item keeps collecting, the triggers keep evaluating, and the problem keeps alerting; once the disable grace elapses the triggers are disabled, which stops them evaluating and hides the problem from the frontend — note that Zabbix *hides* a disabled trigger's open problem but does not resolve it, so no recovery event is sent; after the keep period the triggers are deleted and the problem is removed for good. To silence an already-open problem within that first hour — or to get a clean *recovery* rather than a hidden-then-deleted problem — pair the marker with **Suppress → Indefinitely** (see below), or use the threshold option, which recovers on the next evaluation.
- **Permanent — raise the threshold for one group.** Set `{$PBS.SNAPSHOT.AGE.HIGH:"vm/123"}` (and `{$PBS.SNAPSHOT.AGE.WARN:"vm/123"}`) on the host to a value larger than the age will ever reach, e.g. `3650d`. The expression then evaluates to OK and stays recovered — and unlike the marker, this clears an already-open problem on the next trigger evaluation, with a real recovery event and no disable/delete wait at all. Useful if you'd rather not touch PBS, but you manage it per group in Zabbix and must remember to undo it.

**About acknowledging / closing the problem:** acknowledging is informational only and does not stop the alert. Manually closing it (these triggers have *Allow manual close* enabled) does **not** make it stay closed either — the age keeps rising, the expression is still true, and Zabbix generates a fresh problem on the next poll. Manual close is there for the case where the group is about to be pruned away anyway. To silence one *ad hoc* without changing PBS or macros, use Zabbix's **Suppress** → *Indefinitely* on the problem; that mutes notifications and hides it until it actually clears.

If you keep a prune policy and do **not** protect the last snapshot, the situation also self-heals eventually: once prune removes the group's final snapshot it drops out of discovery and, after the rule's lost-resource periods (disabled after `{$PBS.LLD.DISABLE.LOST}`, deleted after `{$PBS.LLD.KEEP.LOST}`), the age item and any leftover problems are removed automatically. The marker/threshold options above are for the case where you deliberately retain the old backups.

### LLD rule Datastore discovery

| Name                     | Description                                                                                       | Type           | Key and additional info |
| ------------------------ | ------------------------------------------------------------------------------------------------ | -------------- | ----------------------- |
| PBS: Datastore discovery | <p>Discovers the datastores configured on the Proxmox Backup Server from the datastore-usage endpoint.</p> | Dependent item | pbs.datastore.discovery |

### Item prototypes for Datastore discovery

| Name                                                        | Description                                                                                                                                                                                                                                                                                           | Type           | Key and additional info                                                                                                                                                                                                                                                                                                                            |
| ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PBS: Datastore [{#DATASTORE.NAME}] Available Size           | <p>Available size of datastore in bytes.</p>                                                                                                                                                                                                                                                          | Dependent item | pbs.datastore.available[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].avail.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>                                                                                                                                  |
| PBS: Datastore [{#DATASTORE.NAME}] Error                    | <p>Error status of datastore.</p>                                                                                                                                                                                                                                                                     | Dependent item | pbs.datastore.error[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].error.first()`</p><p>⛔️Custom on fail: Set value to: `No Error`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>                                                                                      |
| PBS: Datastore [{#DATASTORE.NAME}] Estimated Full Date      | <p>Estimation of the Date Time Stamp when the storage will be full. It's calculated via a simple Linear Regression (Least Squares) over the RRD data of the last Month. Missing if not enough data points are available yet. An estimate in the past means that usage is declining or not changing.</p> | Dependent item | pbs.datastore.estimatedfulldate[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].['estimated-full-date'].first()`</p><p>⛔️Custom on fail: Discard value</p></li><li><p>JavaScript: converts the epoch to a date, or `Never`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>                                       |
| PBS: Datastore [{#DATASTORE.NAME}] Estimated Seconds to Full | <p>Estimated number of seconds until the datastore is full. It's calculated via a simple Linear Regression (Least Squares) over the RRD data of the last Month. Missing if not enough data points are available yet. An estimate in the past means that usage is declining or not changing.</p>      | Dependent item | pbs.datastore.estimatedsecondstofull[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].['estimated-full-date'].first()`</p><p>⛔️Custom on fail: Discard value</p></li><li><p>JavaScript: seconds until full, clamped to `0`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>                                       |
| PBS: Datastore [{#DATASTORE.NAME}] Estimated Time to Full   | <p>Estimation of the Time until the storage will be full. It's calculated via a simple Linear Regression (Least Squares) over the RRD data of the last Month. Missing if not enough data points are available yet. An estimate in the past means that usage is declining or not changing.</p>           | Dependent item | pbs.datastore.estimatedtimetofull[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].['estimated-full-date'].first()`</p><p>⛔️Custom on fail: Discard value</p></li><li><p>JavaScript: formats the remaining time as `Days HH:MM:SS`, or `Never`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>                    |
| PBS: Datastore [{#DATASTORE.NAME}] Total Size               | <p>Total size of datastore in bytes.</p>                                                                                                                                                                                                                                                              | Dependent item | pbs.datastore.total[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].total.first()`</p></li></ul>                                                                                                                                                                                            |
| PBS: Datastore [{#DATASTORE.NAME}] Used Size                | <p>Used size of datastore in bytes.</p>                                                                                                                                                                                                                                                                | Dependent item | pbs.datastore.used[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].used.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>                                                                                                                                        |

### Trigger prototypes for Datastore discovery

| Name                                                           | Description                                                                                                            | Expression                                                                                                                                                                                                | Severity | Dependencies and additional info |
| -------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | -------------------------------- |
| PBS: Datastore [{#DATASTORE.NAME}] Available Size              | <p>Datastore [{#DATASTORE.NAME}] has less than {$PBS.DATASTORE.AVAILABLE.MIN:"{#DATASTORE.NAME}"} bytes available.</p> | `min(/Proxmox Backup Server by HTTP/pbs.datastore.available[{#DATASTORE.NAME}],15m)<{$PBS.DATASTORE.AVAILABLE.MIN:"{#DATASTORE.NAME}"}`                                                                   | High     | <p>**Depends on:**</p><ul><li>PBS: API service not available</li></ul> |
| PBS: Datastore [{#DATASTORE.NAME}] Error                       | <p>Datastore [{#DATASTORE.NAME}] is reporting an error!</p>                                                            | `find(/Proxmox Backup Server by HTTP/pbs.datastore.error[{#DATASTORE.NAME}],,"like","No Error")=0`                                                                                                        | High     | <p>**Depends on:**</p><ul><li>PBS: API service not available</li></ul> |
| PBS: Datastore [{#DATASTORE.NAME}] filling up within one month | <p>Datastore [{#DATASTORE.NAME}] is projected to fill within one month.</p>                                            | `max(/Proxmox Backup Server by HTTP/pbs.datastore.estimatedsecondstofull[{#DATASTORE.NAME}],15m)<2419200 and min(/Proxmox Backup Server by HTTP/pbs.datastore.estimatedsecondstofull[{#DATASTORE.NAME}],15m)>0` | Warning  | <p>**Depends on:**</p><ul><li>PBS: API service not available</li><li>PBS: Datastore [{#DATASTORE.NAME}] filling up within one week</li></ul> |
| PBS: Datastore [{#DATASTORE.NAME}] filling up within one week  | <p>Datastore [{#DATASTORE.NAME}] is projected to fill within one week.</p>                                             | `max(/Proxmox Backup Server by HTTP/pbs.datastore.estimatedsecondstofull[{#DATASTORE.NAME}],15m)<604800 and min(/Proxmox Backup Server by HTTP/pbs.datastore.estimatedsecondstofull[{#DATASTORE.NAME}],15m)>0`  | High     | <p>**Depends on:**</p><ul><li>PBS: API service not available</li></ul> |

### LLD rule Disk discovery

| Name                | Description                                                                                            | Type           | Key and additional info |
| ------------------- | ----------------------------------------------------------------------------------------------------- | -------------- | ----------------------- |
| PBS: Disk discovery | <p>Discovers the physical disks reported by the Proxmox Backup Server node from the disk list endpoint.</p> | Dependent item | pbs.disk.discovery      |

### Item prototypes for Disk discovery

| Name                            | Description                                               | Type           | Key and additional info                                                                                                                                                                                |
| ------------------------------- | -------------------------------------------------------- | -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PBS: Disk [{#DISK.PATH}] Model  | <p>Model of the disk.</p>                                | Dependent item | pbs.disk.model[{#DISK.PATH}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.devpath == '{#DISK.PATH}')].model.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>        |
| PBS: Disk [{#DISK.PATH}] Name   | <p>Name of the disk.</p>                                 | Dependent item | pbs.disk.name[{#DISK.PATH}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.devpath == '{#DISK.PATH}')].name.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>          |
| PBS: Disk [{#DISK.PATH}] Serial | <p>Serial of the disk.</p>                               | Dependent item | pbs.disk.serial[{#DISK.PATH}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.devpath == '{#DISK.PATH}')].serial.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>      |
| PBS: Disk [{#DISK.PATH}] Size   | <p>Total Size of disk in bytes.</p>                      | Dependent item | pbs.disk.size[{#DISK.PATH}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.devpath == '{#DISK.PATH}')].size.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>          |
| PBS: Disk [{#DISK.PATH}] Status | <p>Disk status.</p>                                      | Dependent item | pbs.disk.status[{#DISK.PATH}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.devpath == '{#DISK.PATH}')].status.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>      |
| PBS: Disk [{#DISK.PATH}] Used   | <p>Indicates where (and if) that disk is used by PBS.</p> | Dependent item | pbs.disk.used[{#DISK.PATH}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.devpath == '{#DISK.PATH}')].used.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>          |
| PBS: Disk [{#DISK.PATH}] Vendor | <p>Vendor of the disk.</p>                               | Dependent item | pbs.disk.vendor[{#DISK.PATH}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.devpath == '{#DISK.PATH}')].vendor.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>      |
| PBS: Disk [{#DISK.PATH}] Wearout | <p>Disk wearout reported by PBS, as a percentage. The direction (whether a higher number means more or less wear) is SSD-model dependent and not standardized across vendors. Discarded for disks that do not report numeric wearout data.</p> | Dependent item | pbs.disk.wearout[{#DISK.PATH}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.devpath == '{#DISK.PATH}')].wearout.first()`</p></li><li><p>JavaScript: parse to a number; discard the value if non-numeric</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul> |

> **Discovery override:** `PBS: Disk [{#DISK.PATH}] Wearout` is only discovered for disks that report a numeric wearout value (e.g. SSDs). Disks that report no wearout (typically HDDs) are excluded from this one item prototype via an LLD override (`{#DISK.WEAROUT}` not matching `^[0-9.]+$`), so they do not create an empty "no data" wearout item.

### Trigger prototypes for Disk discovery

| Name                                            | Description | Expression                                                                                                                                     | Severity | Dependencies and additional info |
| ----------------------------------------------- | ----------- | -------------------------------------------------------------------------------------------------------------------------------------------- | -------- | -------------------------------- |
| PBS: Disk [{#DISK.PATH}] Model has changed      | <p>The disk model reported for this device path has changed, which usually indicates the physical disk was replaced.</p>                  | `last(/Proxmox Backup Server by HTTP/pbs.disk.model[{#DISK.PATH}],#1)<>last(/Proxmox Backup Server by HTTP/pbs.disk.model[{#DISK.PATH}],#2)`   | Warning  | <p>Manual close: `YES`</p><p>**Depends on:**</p><ul><li>PBS: API service not available</li></ul> |
| PBS: Disk [{#DISK.PATH}] Serial has changed     | <p>The disk serial number reported for this device path has changed, which usually indicates the physical disk was replaced.</p> | `last(/Proxmox Backup Server by HTTP/pbs.disk.serial[{#DISK.PATH}],#1)<>last(/Proxmox Backup Server by HTTP/pbs.disk.serial[{#DISK.PATH}],#2)` | Warning  | <p>Manual close: `YES`</p><p>**Depends on:**</p><ul><li>PBS: API service not available</li></ul> |
| PBS: Disk [{#DISK.PATH}] Status indicates a problem | <p>The disk SMART health status reported by PBS is "failed".</p> | `find(/Proxmox Backup Server by HTTP/pbs.disk.status[{#DISK.PATH}],,"like","failed")=1`                                                       | Warning  | <p>Manual close: `YES`</p><p>**Depends on:**</p><ul><li>PBS: API service not available</li></ul> |
| PBS: Disk [{#DISK.PATH}] Used has changed       | <p>The usage of this disk reported by PBS has changed, for example it was added to or removed from a datastore.</p>      | `last(/Proxmox Backup Server by HTTP/pbs.disk.used[{#DISK.PATH}],#1)<>last(/Proxmox Backup Server by HTTP/pbs.disk.used[{#DISK.PATH}],#2)`     | Warning  | <p>Manual close: `YES`</p><p>**Depends on:**</p><ul><li>PBS: API service not available</li></ul> |

## Value maps

### HTTP response status code

| Value | Mapped to             |
| ----- | --------------------- |
| 200   | OK                    |
| 301   | Moved Permanently     |
| 400   | Bad Request           |
| 401   | Unauthorized          |
| 403   | Forbidden             |
| 404   | Not Found             |
| 405   | Method Not Allowed    |
| 500   | Internal Server Error |
| 502   | Bad Gateway           |
| 503   | Service Unavailable   |
| 504   | Gateway Timeout       |
| 520   | Unknown Error         |
