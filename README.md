# zabbix_truenas_SCALE_snmp
Zabbix monitoring template for truenas scale 25.10.4 &amp; higher

## Overview
Template for monitoring TrueNAS SCALE by SNMP. Based on the original of: https://git.zabbix.com/projects/ZBX/repos/zabbix/browse/templates/app/truenas_snmp?at=release/6.0
The version combination of zabbix & truenas has been evolving over the last few months. I am not able to test backwards compatibility.


As per the comment in this issue: [https://ixsystems.atlassian.net/browse/NAS-120434](https://ixsystems.atlassian.net/browse/NAS-120434?focusedCommentId=181336), Truenas SCALE changed a lot of SNMP and thus OID names in version 22.12.1. This resulted in many broken monitoring data.

I tried reaching out via the forum & reddit to get this resolved, but never got any support or interest in this. I decided to work on this for personal usage & hope to help someone else with this too.

Mid 2026 the community made me aware that truenas had been updated and was shipping with an updated MIBS file. Based on this, I was able to make some updates to monitor Dataset, pools & volumes again.

**NOTE:** I had to delete certain logic in this, since with the new SNMP data the pools no longer share data usage statistics. I had to monitor this on datasets using some calculated fields and the available data. I have built a rule to only do this on the "root" datasets (so no sub datasets/folders) as my logic would only work for those based on the available fields in SNMP I could find. L2ARC and ZIL items are included, but since I am not using truenas in a professional context (no cache/SLOG vdev to test against), those counters simply read zero on my setup and I could not fully validate them.

> I am open for pull requests if others are using more complex setups and can find a working solution for their set-up.

### What works in the latest version
On top of the standard monitoring like CPU, Memory, ARC, interfaces, ..., this template adds the following on top of the [original Zabbix TrueNAS template](https://git.zabbix.com/projects/ZBX/repos/zabbix/browse/templates/app/truenas_snmp?at=release/6.0):
- [x] Monitoring of a ZPool (Health with status triggers, Read operations rate, Read rate, Write operations rate, Write rate)
- [x] Monitoring of a Dataset (Available space, Total space, Usage in %, Used space, high/very-high usage triggers, space-usage graphs)
- [x] Monitoring of a ZVOL (Available space, Used Space, Written space)
- [x] Extended ARC monitoring (ARC size vs target, data/metadata split — "ARC composition")
- [x] L2ARC monitoring (hits, misses, read rate, write rate, size, plus throughput & efficiency graphs)
- [x] ZIL operations monitoring (1s / 5s / 10s windows)
- [x] SNMP alert traps with TrueNAS alert level → Zabbix severity mapping
- [x] Updated to the TrueNAS SCALE 25.10 SNMP MIB (new dataset/zvol/ZIL OIDs)

![ZFS Monitoring](img/ZFS-Monitoring.png)

## Requirements
Zabbix version: 7.0 and higher.

## Tested versions
This template has been tested on:

- TrueNAS-25.10.4 with Zabbix 7.0

## TrueNAS SCALE 25.10 SNMP MIB changes

The TRUENAS-MIB (formerly named FREENAS-MIB) changed significantly in TrueNAS SCALE 25.10:

| Feature | Old OID (≤25.04) | New OID (25.10+) |
|---------|-----------------|-----------------|
| ZFS datasets table | `.1.3.6.1.4.1.50536.1.2.1.1` | `.1.3.6.1.4.1.50536.1.6.1.1` |
| L2ARC (hits/misses/read/write/size) | — | `.1.3.6.1.4.1.50536.1.4.[1..5].0` |
| ZIL ops (1s/5s/10s) | `.1.3.6.1.4.1.50536.1.6.[1,2,3].0` | `.1.3.6.1.4.1.50536.1.5.[1,2,3].0` |
| ZFS volumes (zvols) | `.1.3.6.1.4.1.50536.1.3.1.1` | `.1.3.6.1.4.1.50536.1.2.1.1` |

The ZFS volumes (zvol) table still exists in TrueNAS SCALE 25.10 (verified by snmpwalk) at `.1.3.6.1.4.1.50536.1.2.1.1`, so the ZFS volumes discovery rule remains enabled.

## ZFS cache/log graphs
In addition to the ARC hit ratio graph, the template provides graphs for the rest of the ZFS caching/logging subsystem, also shown on the **ZFS** page of the *TrueNAS: Overview* dashboard:

- **ARC composition** — ARC size vs target (c), and the data/metadata split.
- **L2ARC throughput** — read/write rate from the L2 cache.
- **L2ARC efficiency** — L2ARC hits vs misses.
- **ZIL operations** — zilstat ops over 1s/5s/10s windows.

The L2ARC and ZIL graphs only show meaningful data when a cache and/or log (SLOG) vdev is present; otherwise the underlying counters read zero.


## Setup
- Import the template into Zabbix.
- Enable SNMP daemon at Services in TrueNAS web interface: https://www.truenas.com/docs/core/uireference/services/snmpscreen/
- Link the template to the host.

## SNMP alert traps (optional)
The template includes a `TrueNAS: SNMP alert trap` item that captures TrueNAS alert notifications sent under `.1.3.6.1.4.1.50536.2`, with triggers that map the alert level to a Zabbix severity:

| alertLevel | Zabbix severity |
|---|---|
| error / critical / alert / emergency (4-7) | High |
| warning (3) | Warning |
| info / notice (1-2) | Info |

To use it:
- Configure TrueNAS to send SNMP traps for alerts to your Zabbix server/proxy.
- Configure an SNMP trap receiver in Zabbix (`snmptrapd` + the bundled trap handler).
- For named alert levels, load `TRUENAS-MIB` on the trap host; otherwise the numeric levels (1-7) are matched as a fallback.

The trap triggers use manual close, since traps are stateless events. If your trap-receiver output formats the `alertLevel` varbind differently than `INTEGER: <level>`, adjust the trigger regexps accordingly.

## Upgrades
- If you upgrade from the older v 22.x template, you might wish to remove
  discovered items who no longer match, as the same OID are used for
  other values.
  Autodiscovery will then recreate them correctly in the next run (1h) interval

### Dataset discovery
- All discovered datasets (root and sub-datasets) get the full metric set: used, available, total (calculated), usage % (calculated), the high/very-high usage triggers, and the space-usage graphs.
- Usage % is only meaningful for datasets with a quota/refquota; unquota'd sub-datasets report the pool's free space as `available`, so they read ~0 % and do not trigger.
- Use `{$DATASET.NAME.MATCHES}` (default `.+`) and `{$DATASET.NAME.NOT_MATCHES}` (default `^(boot|.+\.system(.+)?$)`) to include/exclude datasets from discovery.
