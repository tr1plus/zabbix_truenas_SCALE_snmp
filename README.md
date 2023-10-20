# zabbix_truenas_SCALE_snmp
Zabbix monitoring template for truenas scale 22.12.1 &amp; higher

## Overview
Template for monitoring TrueNAS SCALE by SNMP. Based on the original of: https://git.zabbix.com/projects/ZBX/repos/zabbix/browse/templates/app/truenas_snmp?at=release/6.0


As per the comment in this issue: [https://ixsystems.atlassian.net/browse/NAS-120434](https://ixsystems.atlassian.net/browse/NAS-120434?focusedCommentId=181336), Truenas SCALE changed a lot of SNMP and thus OID names in version 22.12.1. This resulted in many broken monitoring data.

I tried reaching out via the forum & reddit to get this resolved, but never got any support or interest in this. I decided to work on this for personal usage & hope to help someone else with this too.

**NOTE:** I had to delete certain logic in this, since with the new SNMP data the pools no longer share data usage statistics. I had to monitor this on datasets using some calculated fields and the available data. I have built a rule to only do this on the "root" datasets (so no sub datasets/folders) as my logic would only work for those based on the available fields in SNMP I could find.

## Requirements
Zabbix version: 6.0 and higher.

## Tested versions
This template has been tested on:

TrueNAS-SCALE-22.12.3.3

## Setup
- Import the template into Zabbix.
- Enable SNMP daemon at Services in TrueNAS web interface: https://www.truenas.com/docs/core/uireference/services/snmpscreen/
- Link the template to the host.

### Additional macro
- {$DATASET.ROOT} was added with default value `^(.+\/(.+))` to ensure only the root dataset would get marked for the item prototype.
