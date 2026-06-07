# Proxmox Backup Server by HTTP

## Overview

Template for monitoring Proxmox Backup Server (PBS) over its REST API using the Zabbix HTTP agent.

It collects the API service status, datastore usage (including a linear-regression estimate of when each datastore will be full), failed/erroneous tasks and the physical disk inventory with its SMART status.

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

### Items

| Name                      | Description                       | Type       | Key and additional info                                                                                                                                                                                                                                              |
| ------------------------- | --------------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| PBS: API service status   | <p>Get API service status.</p>    | Script     | pbs.api.available<p>**Preprocessing**</p><ul><li><p>Discard unchanged with heartbeat: `12h`</p></li></ul><p>Value map: `HTTP response status code`</p>                                                                                                              |
| PBS: Get datastore status | <p>Get datastore status.</p>      | HTTP agent | pbs.datastore.status<p>**Preprocessing**</p><ul><li><p>Check for not supported value</p><p>⛔️Custom on fail: Set value to: `Error getting data`</p></li><li><p>JSONPath: `$.body.data`</p></li></ul>                                                                |
| PBS: Get disks            | <p>Get disks.</p>                 | HTTP agent | pbs.disks<p>**Preprocessing**</p><ul><li><p>Check for not supported value</p><p>⛔️Custom on fail: Set value to: `Error getting data`</p></li><li><p>JSONPath: `$.body.data`</p></li></ul>                                                                            |
| PBS: Get failed tasks     | <p>Get erroneuous tasks.</p>      | HTTP agent | pbs.tasks.error<p>**Preprocessing**</p><ul><li><p>Check for not supported value</p><p>⛔️Custom on fail: Set value to: `Error getting data`</p></li><li><p>JSONPath: `$.body.data`</p></li><li><p>JavaScript: filters tasks newer than `{$PBS.TASKS.DAYS}` days</p></li></ul> |

### Triggers

| Name                           | Description                                                                                | Expression                                                  | Severity | Dependencies and additional info |
| ------------------------------ | ----------------------------------------------------------------------------------------- | ---------------------------------------------------------- | -------- | -------------------------------- |
| PBS: API service not available | <p>The API service is not available. Check the network connection, the PBS service status and the API token authorization settings.</p> | `last(/Proxmox Backup Server by HTTP/pbs.api.available)<>200` | High     |                                  |
| PBS: Failed tasks found        | <p>Erroneus tasks that occured within the last {$PBS.TASKS.DAYS} days have been found.</p> | `last(/Proxmox Backup Server by HTTP/pbs.tasks.error)<>"[]"`  | High     | <p>**Depends on:**</p><ul><li>PBS: API service not available</li></ul> |

### LLD rule Datastore discovery

| Name                     | Description                                                                                       | Type           | Key and additional info |
| ------------------------ | ------------------------------------------------------------------------------------------------ | -------------- | ----------------------- |
| PBS: Datastore discovery | <p>Discovers the datastores configured on the Proxmox Backup Server from the datastore-usage endpoint.</p> | Dependent item | pbs.datastore.discovery |

### Item prototypes for Datastore discovery

| Name                                                        | Description                                                                                                                                                                                                                                                                                           | Type           | Key and additional info                                                                                                                                                                                                                                                                                                                            |
| ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PBS: Datastore [{#DATASTORE.NAME}] Available Size           | <p>Available size of datastore in bytes.</p>                                                                                                                                                                                                                                                          | Dependent item | pbs.datastore.available[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].avail.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>                                                                                                                                  |
| PBS: Datastore [{#DATASTORE.NAME}] Error                    | <p>Error status of datastore.</p>                                                                                                                                                                                                                                                                     | Dependent item | pbs.datastore.error[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].error.first()`</p><p>⛔️Custom on fail: Set value to: `No Error`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>                                                                                      |
| PBS: Datastore [{#DATASTORE.NAME}] Estimated Full Date      | <p>Estimation of the Date Time Stamp when the storage will be full. It's calculated via a simple Linear Regression (Least Squares) over the RRD data of the last Month. Missing if not enough data points are available yet. An estimate in the past means that usage is declining or not changing.</p> | Dependent item | pbs.datastore.estimatedfulldate[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].['estimated-full-date'].first()`</p></li><li><p>JavaScript: converts the epoch to a date, or `Never`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>                                       |
| PBS: Datastore [{#DATASTORE.NAME}] Estimated Seconds to Full | <p>Estimated number of seconds until the datastore is full. It's calculated via a simple Linear Regression (Least Squares) over the RRD data of the last Month. Missing if not enough data points are available yet. An estimate in the past means that usage is declining or not changing.</p>      | Dependent item | pbs.datastore.estimatedsecondstofull[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].['estimated-full-date'].first()`</p></li><li><p>JavaScript: seconds until full, clamped to `0`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>                                       |
| PBS: Datastore [{#DATASTORE.NAME}] Estimated Time to Full   | <p>Estimation of the Time until the storage will be full. It's calculated via a simple Linear Regression (Least Squares) over the RRD data of the last Month. Missing if not enough data points are available yet. An estimate in the past means that usage is declining or not changing.</p>           | Dependent item | pbs.datastore.estimatedtimetofull[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].['estimated-full-date'].first()`</p></li><li><p>JavaScript: formats the remaining time as `Days HH:MM:SS`, or `Never`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>                    |
| PBS: Datastore [{#DATASTORE.NAME}] Total Size               | <p>Total size of datastore in bytes.</p>                                                                                                                                                                                                                                                              | Dependent item | pbs.datastore.total[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].total.first()`</p></li></ul>                                                                                                                                                                                            |
| PBS: Datastore [{#DATASTORE.NAME}] Used Size                | <p>Used size of datastore in bytes.</p>                                                                                                                                                                                                                                                                | Dependent item | pbs.datastore.used[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].used.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>                                                                                                                                        |

### Trigger prototypes for Datastore discovery

| Name                                                           | Description                                                                                                            | Expression                                                                                                                                                                                                | Severity | Dependencies and additional info |
| -------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | -------------------------------- |
| PBS: Datastore [{#DATASTORE.NAME}] Available Size              | <p>Datastore [{#DATASTORE.NAME}] has less than {$PBS.DATASTORE.AVAILABLE.MIN:"{#DATASTORE.NAME}"} bytes available.</p> | `min(/Proxmox Backup Server by HTTP/pbs.datastore.available[{#DATASTORE.NAME}],15m)<{$PBS.DATASTORE.AVAILABLE.MIN:"{#DATASTORE.NAME}"}`                                                                   | High     | <p>**Depends on:**</p><ul><li>PBS: API service not available</li></ul> |
| PBS: Datastore [{#DATASTORE.NAME}] Error                       | <p>Datastore [{#DATASTORE.NAME}] is reporting an error!</p>                                                            | `find(/Proxmox Backup Server by HTTP/pbs.datastore.error[{#DATASTORE.NAME}],,"like","No Error")=0`                                                                                                        | High     | <p>**Depends on:**</p><ul><li>PBS: API service not available</li></ul> |
| PBS: Datastore [{#DATASTORE.NAME}] filling up within one month | <p>Datastore [{#DATASTORE.NAME}] is filling up.</p>                                                                    | `last(/Proxmox Backup Server by HTTP/pbs.datastore.estimatedsecondstofull[{#DATASTORE.NAME}])<2419200 and last(/Proxmox Backup Server by HTTP/pbs.datastore.estimatedsecondstofull[{#DATASTORE.NAME}])>0` | High     | <p>**Depends on:**</p><ul><li>PBS: API service not available</li></ul> |
| PBS: Datastore [{#DATASTORE.NAME}] filling up within one week  | <p>Datastore [{#DATASTORE.NAME}] is filling up.</p>                                                                    | `last(/Proxmox Backup Server by HTTP/pbs.datastore.estimatedsecondstofull[{#DATASTORE.NAME}])<604800 and last(/Proxmox Backup Server by HTTP/pbs.datastore.estimatedsecondstofull[{#DATASTORE.NAME}])>0`  | High     | <p>**Depends on:**</p><ul><li>PBS: API service not available</li></ul> |

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
