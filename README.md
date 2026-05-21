# Proxmox Backup Server by HTTP

## Overview

Template for monitoring Proxmox Backup Server (PBS) via its HTTP API.

The template collects datastore usage, disk health and wearout, and failed task information through the PBS REST API and discovers datastores and disks via low-level discovery.

PBS uses an API, the documentation can be found here: https://pbs.proxmox.com/docs/api-viewer/index.html

## Requirements

Zabbix version: 7.2 and higher.

## Tested versions

This template has been tested on Proxmox Backup Server 3.1..3.4.1.

## Author

nikosch86

Source and issue tracker: https://github.com/nikosch86/zabbix-proxmox-backup-server

## Setup

Create a monitoring user and a corresponding API Token.

Set the following access levels for the User and the Token:

- Check: ["perm","/",["Audit"]]

Use the resulting Token ID and Secret in the host macros `{$PBS.TOKEN.ID}` and `{$PBS.TOKEN.SECRET}`, and set `{$PBS.HOST}` to the address of the PBS server.

### Macros used

| Name | Description | Default |
| ---- | ----------- | ------- |
| {$PBS.HOST} | <p>Hostname or IP address of the PBS server API endpoint. Must be set on the host.</p> | `<SET PBS HOSTNAME>` |
| {$PBS.PORT} | <p>The API uses the HTTPS protocol and the server listens on port 8007 by default.</p> | `8007` |
| {$PBS.TOKEN.ID} | <p>API tokens allow stateless access to most parts of the REST API by another system, software or API client.</p> | `USER@REALM!TOKENID` |
| {$PBS.TOKEN.SECRET} | <p>Secret key of the API token. Configured as a secret macro.</p> | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| {$PBS.DATASTORE.AVAILABLE.MIN} | <p>Minimum available space in a datastore, in bytes (defaults to 10 GB). Can be overridden per datastore using the `{#DATASTORE.NAME}` macro context.</p> | `10737418240` |
| {$PBS.TASKS.DAYS} | <p>Age of tasks to consider when looking for failed tasks (days).</p> | `2` |

### Items

| Name | Description | Type | Key and additional info |
| ---- | ----------- | ---- | ----------------------- |
| PBS: API service status | <p>Get API service status.</p> | Script | pbs.api.available<p>Update: 5m</p><p>**Preprocessing**</p><ul><li><p>Discard unchanged with heartbeat: `12h`</p></li></ul><p>**Value map**: HTTP response status code</p> |
| PBS: Get datastore status | <p>Get datastore status.</p> | HTTP agent | pbs.datastore.status<p>Update: 5m</p><p>**Preprocessing**</p><ul><li><p>Check for not supported value</p><p>⛔️Custom on fail: Set value to: `Error getting data`</p></li><li><p>JSON Path: `$.body.data`</p></li></ul> |
| PBS: Get disks | <p>Get disks.</p> | HTTP agent | pbs.disks<p>Update: 5m</p><p>**Preprocessing**</p><ul><li><p>Check for not supported value</p><p>⛔️Custom on fail: Set value to: `Error getting data`</p></li><li><p>JSON Path: `$.body.data`</p></li></ul> |
| PBS: Get failed tasks | <p>Get erroneuous tasks.</p> | HTTP agent | pbs.tasks.error<p>Update: 5m</p><p>**Preprocessing**</p><ul><li><p>Check for not supported value</p><p>⛔️Custom on fail: Set value to: `Error getting data`</p></li><li><p>JSON Path: `$.body.data`</p></li><li><p>JavaScript: filters tasks newer than `{$PBS.TASKS.DAYS}` days</p></li></ul> |

### Triggers

| Name | Description | Expression | Severity | Dependencies and additional info |
| ---- | ----------- | ---------- | -------- | -------------------------------- |
| PBS: API service not available | <p>The API service is not available. Check the network connection, the PBS service status and the API token authorization settings.</p> | `last(/Proxmox Backup Server by HTTP/pbs.api.available)<>200` | High | |
| PBS: Failed tasks found | <p>Erroneus tasks that occured within the last {$PBS.TASKS.DAYS} days have been found.</p> | `last(/Proxmox Backup Server by HTTP/pbs.tasks.error)<>"[]"` | High | <p>Recovery expression: `last(/Proxmox Backup Server by HTTP/pbs.tasks.error)="[]"`</p><p>**Depends on**:</p><ul><li>PBS: API service not available</li></ul> |

### LLD rule Datastore discovery

| Name | Description | Type | Key and additional info |
| ---- | ----------- | ---- | ----------------------- |
| PBS: Datastore discovery | | Dependent item | pbs.datastore.discovery<p>**LLD macro**: `{#DATASTORE.NAME}` ← `$.store`</p> |

### Item prototypes for Datastore discovery

| Name | Description | Type | Key and additional info |
| ---- | ----------- | ---- | ----------------------- |
| PBS: Datastore [{#DATASTORE.NAME}] Available Size | <p>Available size of datastore in bytes.</p> | Dependent item | pbs.datastore.available[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSON Path: `$.[?(@.store == '{#DATASTORE.NAME}')].avail.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul> |
| PBS: Datastore [{#DATASTORE.NAME}] Error | <p>Error status of datastore.</p> | Dependent item | pbs.datastore.error[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSON Path: `$.[?(@.store == '{#DATASTORE.NAME}')].error.first()`</p><p>⛔️Custom on fail: Set value to: `No Error`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul> |
| PBS: Datastore [{#DATASTORE.NAME}] Estimated Full Date | <p>Estimation of the Date Time Stamp when the storage will be full. It's calculated via a simple Linear Regression (Least Squares) over the RRD data of the last Month. Missing if not enough data points are available yet. An estimate in the past means that usage is declining or not changing.</p> | Dependent item | pbs.datastore.estimatedfulldate[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSON Path: `$.[?(@.store == '{#DATASTORE.NAME}')].['estimated-full-date'].first()`</p></li><li><p>JavaScript: returns the full date, or `Never` if the estimate is in the past</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul> |
| PBS: Datastore [{#DATASTORE.NAME}] Estimated Seconds to Full | <p>Estimation of the UNIX epoch when the storage will be full. It's calculated via a simple Linear Regression (Least Squares) over the RRD data of the last Month. Missing if not enough data points are available yet. An estimate in the past means that usage is declining or not changing.</p> | Dependent item | pbs.datastore.estimatedsecondstofull[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSON Path: `$.[?(@.store == '{#DATASTORE.NAME}')].['estimated-full-date'].first()`</p></li><li><p>JavaScript: returns the number of seconds until full (0 if in the past)</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul> |
| PBS: Datastore [{#DATASTORE.NAME}] Estimated Time to Full | <p>Estimation of the Time until the storage will be full. It's calculated via a simple Linear Regression (Least Squares) over the RRD data of the last Month. Missing if not enough data points are available yet. An estimate in the past means that usage is declining or not changing.</p> | Dependent item | pbs.datastore.estimatedtimetofull[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSON Path: `$.[?(@.store == '{#DATASTORE.NAME}')].['estimated-full-date'].first()`</p></li><li><p>JavaScript: returns a human readable `Days HH:MM:SS` string, or `Never` if the estimate is in the past</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul> |
| PBS: Datastore [{#DATASTORE.NAME}] Total Size | <p>Total size of datastore in bytes.</p> | Dependent item | pbs.datastore.total[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSON Path: `$.[?(@.store == '{#DATASTORE.NAME}')].total.first()`</p></li></ul> |
| PBS: Datastore [{#DATASTORE.NAME}] Used Size | <p>Used size of datastore in bytes.</p> | Dependent item | pbs.datastore.used[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSON Path: `$.[?(@.store == '{#DATASTORE.NAME}')].used.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul> |

### Trigger prototypes for Datastore discovery

| Name | Description | Expression | Severity | Dependencies and additional info |
| ---- | ----------- | ---------- | -------- | -------------------------------- |
| PBS: Datastore [{#DATASTORE.NAME}] Available Size | <p>Datastore [{#DATASTORE.NAME}] has less than {$PBS.DATASTORE.AVAILABLE.MIN:"{#DATASTORE.NAME}"} bytes available.</p> | `min(/Proxmox Backup Server by HTTP/pbs.datastore.available[{#DATASTORE.NAME}],15m)<{$PBS.DATASTORE.AVAILABLE.MIN:"{#DATASTORE.NAME}"}` | High | <p>**Depends on**:</p><ul><li>PBS: API service not available</li></ul> |
| PBS: Datastore [{#DATASTORE.NAME}] Error | <p>Datastore [{#DATASTORE.NAME}] is reporting an error!</p> | `find(/Proxmox Backup Server by HTTP/pbs.datastore.error[{#DATASTORE.NAME}],,"like","No Error")=0` | High | <p>**Depends on**:</p><ul><li>PBS: API service not available</li></ul> |
| PBS: Datastore [{#DATASTORE.NAME}] filling up within one month | <p>Datastore [{#DATASTORE.NAME}] is filling up.</p> | `last(/Proxmox Backup Server by HTTP/pbs.datastore.estimatedsecondstofull[{#DATASTORE.NAME}])<2419200 and last(/Proxmox Backup Server by HTTP/pbs.datastore.estimatedsecondstofull[{#DATASTORE.NAME}])>0` | High | <p>**Depends on**:</p><ul><li>PBS: API service not available</li></ul> |
| PBS: Datastore [{#DATASTORE.NAME}] filling up within one week | <p>Datastore [{#DATASTORE.NAME}] is filling up.</p> | `last(/Proxmox Backup Server by HTTP/pbs.datastore.estimatedsecondstofull[{#DATASTORE.NAME}])<604800 and last(/Proxmox Backup Server by HTTP/pbs.datastore.estimatedsecondstofull[{#DATASTORE.NAME}])>0` | High | <p>**Depends on**:</p><ul><li>PBS: API service not available</li></ul> |

### LLD rule Disk discovery

| Name | Description | Type | Key and additional info |
| ---- | ----------- | ---- | ----------------------- |
| PBS: Disk discovery | | Dependent item | pbs.disk.discovery<p>**LLD macros**: `{#DISK.NAME}`, `{#DISK.PATH}`, `{#DISK.SERIAL}`, `{#DISK.SIZE}`, `{#DISK.STATUS}`, `{#DISK.TYPE}`, `{#DISK.USED}`, `{#DISK.VENDOR}`, `{#DISK.WEAROUT}`</p> |

### Item prototypes for Disk discovery

| Name | Description | Type | Key and additional info |
| ---- | ----------- | ---- | ----------------------- |
| PBS: Disk [{#DISK.PATH}] Model | <p>Model of the disk.</p> | Dependent item | pbs.disk.model[{#DISK.PATH}]<p>**Preprocessing**</p><ul><li><p>JSON Path: `$.[?(@.devpath == '{#DISK.PATH}')].model.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul> |
| PBS: Disk [{#DISK.PATH}] Name | <p>Name of the disk.</p> | Dependent item | pbs.disk.name[{#DISK.PATH}]<p>**Preprocessing**</p><ul><li><p>JSON Path: `$.[?(@.devpath == '{#DISK.PATH}')].name.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul> |
| PBS: Disk [{#DISK.PATH}] Serial | <p>Serial of the disk.</p> | Dependent item | pbs.disk.serial[{#DISK.PATH}]<p>**Preprocessing**</p><ul><li><p>JSON Path: `$.[?(@.devpath == '{#DISK.PATH}')].serial.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul> |
| PBS: Disk [{#DISK.PATH}] Size | <p>Total Size of disk in bytes.</p> | Dependent item | pbs.disk.size[{#DISK.PATH}]<p>**Preprocessing**</p><ul><li><p>JSON Path: `$.[?(@.devpath == '{#DISK.PATH}')].size.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul> |
| PBS: Disk [{#DISK.PATH}] Status | <p>Disk status.</p> | Dependent item | pbs.disk.status[{#DISK.PATH}]<p>**Preprocessing**</p><ul><li><p>JSON Path: `$.[?(@.devpath == '{#DISK.PATH}')].status.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul> |
| PBS: Disk [{#DISK.PATH}] Used | <p>Indicates where (and if) that disk is used by PBS.</p> | Dependent item | pbs.disk.used[{#DISK.PATH}]<p>**Preprocessing**</p><ul><li><p>JSON Path: `$.[?(@.devpath == '{#DISK.PATH}')].used.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul> |
| PBS: Disk [{#DISK.PATH}] Vendor | <p>Vendor of the disk.</p> | Dependent item | pbs.disk.vendor[{#DISK.PATH}]<p>**Preprocessing**</p><ul><li><p>JSON Path: `$.[?(@.devpath == '{#DISK.PATH}')].vendor.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul> |
| PBS: Disk [{#DISK.PATH}] Wearout | <p>Disk wearout reported by PBS, as a percentage. The direction (whether a higher number means more or less wear) is SSD-model dependent and not standardized across vendors. Discarded for disks that report no numeric wearout data.</p> | Dependent item | pbs.disk.wearout[{#DISK.PATH}]<p>**Preprocessing**</p><ul><li><p>JSON Path: `$.[?(@.devpath == '{#DISK.PATH}')].wearout.first()`</p></li><li><p>JavaScript: converts the value to a number; discards it when the disk reports no numeric wearout</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul> |

### Trigger prototypes for Disk discovery

| Name | Description | Expression | Severity | Dependencies and additional info |
| ---- | ----------- | ---------- | -------- | -------------------------------- |
| PBS: Disk [{#DISK.PATH}] Model has changed | <p>The disk model reported for this device path has changed, which usually indicates the physical disk was replaced.</p> | `last(/Proxmox Backup Server by HTTP/pbs.disk.model[{#DISK.PATH}],#1)<>last(/Proxmox Backup Server by HTTP/pbs.disk.model[{#DISK.PATH}],#2)` | Warning | <p>Manual close: `YES`</p><p>**Depends on**:</p><ul><li>PBS: API service not available</li></ul> |
| PBS: Disk [{#DISK.PATH}] Serial has changed | <p>The disk serial number reported for this device path has changed, which usually indicates the physical disk was replaced.</p> | `last(/Proxmox Backup Server by HTTP/pbs.disk.serial[{#DISK.PATH}],#1)<>last(/Proxmox Backup Server by HTTP/pbs.disk.serial[{#DISK.PATH}],#2)` | Warning | <p>Manual close: `YES`</p><p>**Depends on**:</p><ul><li>PBS: API service not available</li></ul> |
| PBS: Disk [{#DISK.PATH}] Status indicates a problem | <p>The disk SMART health status reported by PBS is "failed".</p> | `find(/Proxmox Backup Server by HTTP/pbs.disk.status[{#DISK.PATH}],,"like","failed")=1` | Warning | <p>Manual close: `YES`</p><p>**Depends on**:</p><ul><li>PBS: API service not available</li></ul> |
| PBS: Disk [{#DISK.PATH}] Used has changed | <p>The usage of this disk reported by PBS has changed, for example it was added to or removed from a datastore.</p> | `last(/Proxmox Backup Server by HTTP/pbs.disk.used[{#DISK.PATH}],#1)<>last(/Proxmox Backup Server by HTTP/pbs.disk.used[{#DISK.PATH}],#2)` | Warning | <p>Manual close: `YES`</p><p>**Depends on**:</p><ul><li>PBS: API service not available</li></ul> |

### Value maps

| Name | Mappings |
| ---- | -------- |
| HTTP response status code | 200 ⇒ OK; 301 ⇒ Moved Permanently; 400 ⇒ Bad Request; 401 ⇒ Unauthorized; 403 ⇒ Forbidden; 404 ⇒ Not Found; 405 ⇒ Method Not Allowed; 500 ⇒ Internal Server Error; 502 ⇒ Bad Gateway; 503 ⇒ Service Unavailable; 504 ⇒ Gateway Timeout; 520 ⇒ Unknown Error |
