---
title: "Influxdb Size"
date: 2022-08-03T04:41:58+02:00
draft: false
---

# Reduce Home Assistant InfluxDB size

## Intro 

Lately I was running into issues where the recorder stopped recording events, see [other article]().
Some investigation revealed this was most likely caused by a backup locking the database for too long.

That made me venture into the HA shell and find out where all this space was being consumed.
I'm running full backups every 2 days, keeping 3 backups on HA and some on Google Drive. The backup size was ~11GB now.

## Size lookup

- use ha 'supershell' to port 2222 to be able to see all mounts easily
- inspect your disks, initially with `df -h`, later drilling down with `du -h -d1` in your folders

```bash

# du -h -d1
61.5G	./supervisor
16.0K	./lost+found
^C
# cd supervisor/
# du -h -d1
108.0K	./apparmor
36.0K	./ssl
12.0K	./dns
225.3M	./homeassistant
36.6G	./backup
24.4G	./addons
305.6M	./share
4.0K	./media
52.0K	./audio
5.3M	./tmp
61.5G	.
```

Here we find that indeed our `backup` folder is taking up >30G as mentioned. Ignoring that for now, we see the next biggest one is `addons`.

```bash
# cd addons
# du -h -d1
21.9M	./git
8.1M	./core
24.3G	./data
4.0K	./local
24.4G	.

# cd data
# du -h -d1
292.0K	./core_duckdns
812.0K	./a0d7b954_motioneye
17.6G	./a0d7b954_influxdb
8.0K	./45df7312_zigbee2mqtt
304.0K	./core_mosquitto
2.0M	./a0d7b954_grafana
668.5M	./a0d7b954_unifi
180.2M	./a0d7b954_vscode
12.0K	./3a26b21d_eufy_security_addon
20.0K	./core_nginx_proxy
8.0K	./core_configurator
8.0K	./a0d7b954_glances
4.4G	./core_mariadb
52.0K	./core_ssh
12.0K	./a0d7b954_phpmyadmin
1.5G	./a0d7b954_esphome
8.0K	./75a80a57_rtsp_simple_server
40.0K	./cebe7a76_hassio_google_drive_backup
24.3G	.

```

4.5G for mariadb (recorder), 1.5G for esphome build files; all acceptable to me. But, InfluxDB is eating up 17.6G! 


## InfluxDB size reduction

I was sending basically all HA data to influxdb since the setup ~14 months ago.

Let's revise our strategy here. I was using only the `autogen` DEFAULT retention policy, with an infinite retention.

* reduce data coming into InfluxDB, by excluding data in influxdb HA config
* setup retention policies with shorter periods, with downsampled data

### Reduce HA data sent to InfluxDB

My main usecase for the InfluxDB data is to graph primarily energy data over longer periods.
That is consumption/production of my smart meter, production and self-consumption of my solar power and energy usage of different appliances over the house, etc. Occasionaly (or in the future) I might also want to graph out temerature and humidy data of varous sensors. I'm less interested to know in what state a certain switch or light was 4 months ago.

We can achieve this by excluding entities, entity_globs or even whole domains. I did some checks in my current influxdb and chose to use this setup for now.

```yaml


```


### Setup new retention policies

With different retention policies, combined with continuous queries you can setup downsampling of incoming data with different aggregation levels, and control how long to keep that data. With this setup you can reduce your DEFAULT `autogen` retention policy.

I decided for the following setup:

| Retention Policy | Duration |
|---|---|
| autogen | 60d |
| rp_15min | 52w |
| rp_1h | 52w |
| rp_1d | infinite |

So keep full granularity of incoming data, keeping that for 60 days. That allows me to look closely at something in the full resolution for ~2 months after the facts, that should be plenty.
Downsampled data with 15 min resolution will be kept for 1 year, as will 1 hour resolution data. Data reduced to 1 day will be kept indefinitely.


### Setup continuous queries

To downsample incoming data we use a continuous query. It will write to on the newly defined RP's. As I have created 3 new policies, I'll also need 3 CQ's to fill these when data comes in.


```sql
CREATE CONTINUOUS QUERY "cq_ha_15min" ON "homeassistant"
BEGIN
  SELECT mean(*) INTO "homeassistant"."rp_15min".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ 
  GROUP BY time(15m), *
  FILL(previous)
END

CREATE CONTINUOUS QUERY "cq_ha_1h" ON "homeassistant"
BEGIN
  SELECT mean(*) INTO "homeassistant"."rp_1h".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ 
  GROUP BY time(1h), *
  FILL(previous)
END

CREATE CONTINUOUS QUERY "cq_ha_1d" ON "homeassistant"
BEGIN
  SELECT mean(*) INTO "homeassistant"."rp_1d".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ 
  GROUP BY time(1d), *
  FILL(previous)
END

```

### Backfill data

Continuous queries only act on newly incoming data, and will not handle any data that's already present. If you have historical data in your default RP, you need to backfill the downsampled data to the other RPs.

I have about 14 months worth of data, but only the `rp_1d` data needs to be kept for > 52 weeks.

Because these queries will backfill all measurements at once, it takes quite some resources. To prevent the queries from timing out, I split them in several chuncks. Make sure to run them one by one.

**rp_15min**

```sql
-- Backfill rp_15min
SELECT mean(*) INTO "homeassistant"."rp_15min".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 52w and time < now() - 40w GROUP BY time(15m), * FILL(previous)
SELECT mean(*) INTO "homeassistant"."rp_15min".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 41w and time < now() - 30w GROUP BY time(15m), * FILL(previous)
SELECT mean(*) INTO "homeassistant"."rp_15min".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 31w and time < now() - 20w GROUP BY time(15m), * FILL(previous)
SELECT mean(*) INTO "homeassistant"."rp_15min".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 21w and time < now() - 10w GROUP BY time(15m), * FILL(previous)
SELECT mean(*) INTO "homeassistant"."rp_15min".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 11w and time < now() GROUP BY time(15m), * FILL(previous)

-- Backfill rp_1h

SELECT mean(*) INTO "homeassistant"."rp_1h".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 52w and time < now() - 40w GROUP BY time(1h), * FILL(previous)
SELECT mean(*) INTO "homeassistant"."rp_1h".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 41w and time < now() - 30w GROUP BY time(1h), * FILL(previous)
SELECT mean(*) INTO "homeassistant"."rp_1h".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 31w and time < now() - 20w GROUP BY time(1h), * FILL(previous)
SELECT mean(*) INTO "homeassistant"."rp_1h".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 21w and time < now() - 10w GROUP BY time(1h), * FILL(previous)
-- TODO
SELECT mean(*) INTO "homeassistant"."rp_1h".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 11w and time < now() GROUP BY time(1h), * FILL(previous)


-- Backfill rp_1d
SELECT mean(*) INTO "homeassistant"."rp_1d".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 99w and time < now() - 50w GROUP BY time(1d), * FILL(previous)
SELECT mean(*) INTO "homeassistant"."rp_1d".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 52w and time < now() - 40w GROUP BY time(1d), * FILL(previous)
SELECT mean(*) INTO "homeassistant"."rp_1d".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 41w and time < now() - 30w GROUP BY time(1d), * FILL(previous)
SELECT mean(*) INTO "homeassistant"."rp_1d".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 31w and time < now() - 20w GROUP BY time(1d), * FILL(previous)
SELECT mean(*) INTO "homeassistant"."rp_1d".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 21w and time < now() - 10w GROUP BY time(1d), * FILL(previous)
SELECT mean(*) INTO "homeassistant"."rp_1d".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 11w and time < now() GROUP BY time(1d), * FILL(previous)
