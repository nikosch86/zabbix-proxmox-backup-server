# Proxmox Backup Server

## Overview

Template for monitoring Proxmox Backup Server (PBS) 3.1.

PBS uses an API, the documentation can be found here: https://pbs.proxmox.com/docs/api-viewer/index.html

## Requirements

Zabbix 6.4 and higher

## Tested versions

This template has been tested on Proxmox Backup Server 3.1

## Author

nikosch86

## Setup

Create a monitoring user and a corresponding API Token.

Set the following access levels for the User and the Token:

- Check: ["perm","/",["Audit"]]

Use the resulting Token ID and Secret in the host macros.

### Macros used

| Name                           | Description                                                                                                       | Default                                |
| ------------------------------ | ----------------------------------------------------------------------------------------------------------------- | -------------------------------------- |
| {$PBS.HOST}                | <p>The Hostname of the PBS server. Defaults to the HOST.CONN of the configured host.</p>                            | `{HOST.CONN}`                                 |
| {$PBS.PORT}                | <p>The API uses the HTTPS protocol and the server listens to port 8007 by default.</p>                            | `8007`                                 |
| {$PBS.TOKEN.ID}                | <p>API tokens allow stateless access to most parts of the REST API by another system, software or API client.</p> | `USER@REALM!TOKENID`                   |
| {$PBS.TOKEN.SECRET}            | <p>Secret key.</p>                                                                                                | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| {$PBS.DATASTORE.AVAILABLE.MIN} | <p>Minimum available space in datastore in bytes (defaults to 10Gb).</p>                                          | `10737418240`                          |
| {$PBS.TASKS.DAYS} | <p>Age of tasks to consider when looking for failed tasks (days) (defaults to 2).</p>                                          | `2`                          |

### Items

| Name                      | Description                              | Type       | Key and additional info                                                                                                                                                                               |
| ------------------------- | ---------------------------------------- | ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PBS: Get disks            | <p>Get all disks status.</p>             | HTTP agent | pbs.disks<p>**Preprocessing**</p><ul><li><p>Check for not supported value</p><p>⛔️Custom on fail: Set value to: `Error getting data`</p></li><li><p>JSONPath: `$.body.data`</p></li></ul>            |
| PBS: Get datastore status | <p>Get datastore status information.</p> | HTTP agent | pbs.datastore.status<p>**Preprocessing**</p><ul><li><p>Check for not supported value</p><p>⛔️Custom on fail: Set value to: `Error getting data`</p></li><li><p>JSONPath: `$.body.data`</p></li></ul> |
| PBS: API service status   | <p>Get API service status.</p>           | Script     | pbs.api.available<p>**Preprocessing**</p><ul><li><p>Discard unchanged with heartbeat: `12h`</p></li></ul>                                                                                             |
| PBS: Get failed tasks     | <p>Get erroneuous tasks.</p>             | HTTP agent | pbs.tasks.error<p>**Preprocessing**</p><ul><li><p>Check for not supported value</p><p>⛔️Custom on fail: Set value to: `Error getting data`</p></li><li><p>JSONPath: `$.body.data`</p></li><li><p>JavaScript (filters tasks by age)</p></li></ul>                                                                                             |

### Triggers

| Name                           | Description                                                                             | Expression                                                      | Severity | Dependencies and additional info |
| ------------------------------ | --------------------------------------------------------------------------------------- | --------------------------------------------------------------- | -------- | -------------------------------- |
| PBS: API service not available | <p>The API service is not available. Check your network and authorization settings.</p> | `last(/Proxmox Backup Server by HTTP/pbs.api.available) <> 200` | High     |                                  |
| PBS: Failed tasks found        | <p>Erroneus tasks that occured within the last {$PBS.TASKS.DAYS} days have been found.</p> | `last(/Proxmox Backup Server by HTTP/pbs.tasks.error)<>"[]"` | High     |                                  |

### LLD rule Datastore discovery

| Name                     | Description | Type           | Key and additional info |
| ------------------------ | ----------- | -------------- | ----------------------- |
| PBS: Datastore discovery |             | Dependent item | pbs.datastore.discovery |

### Item prototypes for Datastore discovery

| Name                                                          | Description                                                                                                                                                                                                                                                                                             | Type           | Key and additional info                                                                                                                                                                                                                                                                                                                                                                               |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PBS: Datastore [{#DATASTORE.NAME}]: Total Size                | <p>Total size of datastore in bytes.</p>                                                                                                                                                                                                                                                                | Dependent item | pbs.datastore.total[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].total.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>                                                                                                                                                                                          |
| PBS: Datastore [{#DATASTORE.NAME}]: Used Size                 | <p>Used size of datastore in bytes.</p>                                                                                                                                                                                                                                                                 | Dependent item | pbs.datastore.used[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].used.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>                                                                                                                                                                                            |
| PBS: Datastore [{#DATASTORE.NAME}]: Available Size            | <p>Available size of datastore in bytes.</p>                                                                                                                                                                                                                                                            | Dependent item | pbs.datastore.available[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].avail.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>                                                                                                                                                                                      |
| PBS: Datastore [{#DATASTORE.NAME}]: Error                     | <p>Error status of datastore.</p>                                                                                                                                                                                                                                                                       | Dependent item | pbs.datastore.error[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].error.first()`</p><p>⛔️Custom on fail: Set value to: `No Error`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>                                                                                                                                        |
| PBS: Datastore [{#DATASTORE.NAME}]: Estimated Seconds to Full | <p>Estimation of the UNIX epoch when the storage will be full. It's calculated via a simple Linear Regression (Least Squares) over the RRD data of the last Month. Missing if not enough data points are available yet. An estimate in the past means that usage is declining or not changing.</p>      | Dependent item | pbs.datastore.estimatedsecondstofull[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].['estimated-full-date'].first()`</p></li><li><p>Javascript: `var current_unixtime = Math.floor(Date.now() / 1000); var delta = Math.max(0, (value - current_unixtime)); return delta;`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul> |
| PBS: Datastore [{#DATASTORE.NAME}]: Estimated Full Date       | <p>Estimation of the Date Time Stamp when the storage will be full. It's calculated via a simple Linear Regression (Least Squares) over the RRD data of the last Month. Missing if not enough data points are available yet. An estimate in the past means that usage is declining or not changing.</p> | Dependent item | pbs.datastore.estimatedfulldate[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].['estimated-full-date'].first()`</p></li><li><p>Javascript: `too long, see template`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>                                                                                                        |
| PBS: Datastore [{#DATASTORE.NAME}]: Estimated Time to Full    | <p>Estimation of the Time until the storage will be full. It's calculated via a simple Linear Regression (Least Squares) over the RRD data of the last Month. Missing if not enough data points are available yet. An estimate in the past means that usage is declining or not changing.</p>           | Dependent item | pbs.datastore.estimatedfulldate[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].['estimated-full-date'].first()`</p></li><li><p>Javascript: `too long, see template`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>                                                                                                        |

### Trigger prototypes for Datastore discovery

| Name                                                           | Description                                                                                                            | Expression                                                                                                                                                                                                | Severity | Dependencies and additional info |
| -------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | -------------------------------- |
| PBS: Datastore [{#DATASTORE.NAME}] Available Size              | <p>Datastore [{#DATASTORE.NAME}] has less than {$PBS.DATASTORE.AVAILABLE.MIN:"{#DATASTORE.NAME}"} bytes available.</p> | `min(/Proxmox Backup Server by HTTP/pbs.datastore.available[{#DATASTORE.NAME}],15m)<{$PBS.DATASTORE.AVAILABLE.MIN:"{#DATASTORE.NAME}"}`                                                                   | High     |                                  |
| PBS: Datastore [{#DATASTORE.NAME}] Error                       | <p>Datastore [{#DATASTORE.NAME}] is reporting an error!</p>                                                            | `find(/Proxmox Backup Server by HTTP/pbs.datastore.error[{#DATASTORE.NAME}],,"like","No Error")=0`                                                                                                        | High     |                                  |
| PBS: Datastore [{#DATASTORE.NAME}] filling up within one month | <p>Datastore [{#DATASTORE.NAME}] is filling up.</p>                                                                    | `last(/Proxmox Backup Server by HTTP/pbs.datastore.estimatedsecondstofull[{#DATASTORE.NAME}])<2419200 and last(/Proxmox Backup Server by HTTP/pbs.datastore.estimatedsecondstofull[{#DATASTORE.NAME}])>0` | High     |                                  |
| PBS: Datastore [{#DATASTORE.NAME}] filling up within one week  | <p>Datastore [{#DATASTORE.NAME}] is filling up.</p>                                                                    | `last(/Proxmox Backup Server by HTTP/pbs.datastore.estimatedsecondstofull[{#DATASTORE.NAME}])<604800 and last(/Proxmox Backup Server by HTTP/pbs.datastore.estimatedsecondstofull[{#DATASTORE.NAME}])>0`  | High     |                                  |

### LLD rule Disk discovery

| Name                | Description | Type           | Key and additional info |
| ------------------- | ----------- | -------------- | ----------------------- |
| PBS: Disk discovery |             | Dependent item | pbs.disk.discovery      |

### Item prototypes for Disk discovery

| Name                                   | Description                                               | Type           | Key and additional info                                                                                                                                                                                     |
| -------------------------------------- | --------------------------------------------------------- | -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PBS: Disk [{#DATASTORE.NAME}]: Size    | <p>Total Size of disk in bytes.</p>                       | Dependent item | pbs.disk.size[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.devpath == '{#DISK.PATH}')].size.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>          |
| PBS: Disk [{#DATASTORE.NAME}]: Status  | <p>Disk status.</p>                                       | Dependent item | pbs.disk.status[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].status.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>   |
| PBS: Disk [{#DATASTORE.NAME}]: Name    | <p>Name of the disk.</p>                                  | Dependent item | pbs.disk.name[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].name.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>       |
| PBS: Disk [{#DATASTORE.NAME}]: Vendor  | <p>Vendor of the disk.</p>                                | Dependent item | pbs.disk.vendor[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].vendor.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>   |
| PBS: Disk [{#DATASTORE.NAME}]: Model   | <p>Model of the disk.</p>                                 | Dependent item | pbs.disk.model[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].model.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>     |
| PBS: Disk [{#DATASTORE.NAME}]: Serial  | <p>Serial of the disk.</p>                                | Dependent item | pbs.disk.serial[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].serial.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>   |
| PBS: Disk [{#DATASTORE.NAME}]: Used    | <p>Indicates where (and if) that disk is used by PBS.</p> | Dependent item | pbs.disk.used[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].used.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul>       |
| PBS: Disk [{#DATASTORE.NAME}]: Wearout | <p>Wearout indicator.</p>                                 | Dependent item | pbs.disk.wearout[{#DATASTORE.NAME}]<p>**Preprocessing**</p><ul><li><p>JSONPath: `$.[?(@.store == '{#DATASTORE.NAME}')].wearout.first()`</p></li><li><p>Discard unchanged with heartbeat: `1h`</p></li></ul> |

### Trigger prototypes for Datastore discovery

| Name                                                     | Description                                                        | Expression                                                                                                                                     | Severity | Dependencies and additional info |
| -------------------------------------------------------- | ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- | -------- | -------------------------------- |
| PBS: Disk [{#DATASTORE.NAME}] Model has changed          | <p>The model identifier has changed.</p>                           | `last(/Proxmox Backup Server by HTTP/pbs.disk.model[{#DISK.PATH}],#1)<>last(/Proxmox Backup Server by HTTP/pbs.disk.model[{#DISK.PATH}],#2)`   | Warning  |                                  |
| PBS: Disk [{#DATASTORE.NAME}] Serial has changed         | <p>The Serial number has changed.</p>                              | `last(/Proxmox Backup Server by HTTP/pbs.disk.serial[{#DISK.PATH}],#1)<>last(/Proxmox Backup Server by HTTP/pbs.disk.serial[{#DISK.PATH}],#2)` | Warning  |                                  |
| PBS: Disk [{#DATASTORE.NAME}] Status indicates a problem | <p>The Status indicator shows some different from 'unknown'.</p>   | `find(/Proxmox Backup Server by HTTP/pbs.disk.status[{#DISK.PATH}],,"like","unknown")=0`                                                       | Warning  |                                  |
| PBS: Disk [{#DATASTORE.NAME}] Used has changed           | <p>The disk is being reported as used differently than before.</p> | `last(/Proxmox Backup Server by HTTP/pbs.disk.used[{#DISK.PATH}],#1)<>last(/Proxmox Backup Server by HTTP/pbs.disk.used[{#DISK.PATH}],#2)`     | Warning  |                                  |
