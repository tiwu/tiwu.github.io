---
title: "Influxdb Size"
date: 2022-08-03T04:41:58+02:00
draft: false
resources:
  - name: rp-overview
    src: "rp-overview.png"
    title: RP overview
---

{{< toc >}}

# Reduce Home Assistant InfluxDB size

## Intro 

Lately I was running into issues where the recorder stopped recording events.
Some investigation revealed this was most likely caused by a backup locking the database for too long.

That made me venture into the HA shell and find out where all this space was being consumed.
I'm running full backups every 2 days, keeping 3 backups on HA and some on Google Drive. The backup size was ~11GB now.

## Size lookup

- use HA's [developer console](https://developers.home-assistant.io/docs/operating-system/debugging/#home-assistant-operating-system) to ssh into port 2222 to be able to see all mounts easily. The standard ssh addon does not give you access to the other containers that are running. The developer console is really into the HAOS, and you can inspect the full filesystem of the host machine.
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

4.5G for mariadb (recorder), 1.5G for esphome build files; all acceptable to me. But, InfluxDB is eating up 17.6G! 😮 Let's drill down even further to identify the space per database and retention policy.

```bash
# cd a0d7b954_influxdb/influxdb/data
# du -h -d1
96.1M	./_internal
17.7G	./homeassistant
32.0M	./glances
17.8G	.
# cd homeassistant
# pwd
/mnt/data/supervisor/addons/data/a0d7b954_influxdb/influxdb/data/homeassistant
# du -h -d1
32.0M	./_series
17.6G	./autogen
17.7G	.

```

That confirms, all the space for influxdb is used for our Home Assistant database.

## InfluxDB size reduction

With the default setup I was sending basically all HA data to influxdb since the setup ~14 months ago. Every update of every sensor, switch, light and and all my energy monitors reporting values every few seconds.

I was using only the `autogen` DEFAULT retention policy, with an infinite retention. Thus keeping all data in raw form indefinitely!

After some reading up in the [influxdb](https://docs.influxdata.com/influxdb/v1.8/guides/downsample_and_retain/) and [Home Asisstant](https://www.home-assistant.io/integrations/influxdb/#configure-filter) documentation, came up with the following steps:

* reduce data flowing to InfluxDB, by excluding data in the `influxdb` HA config
* setup retention policies with shorter periods
* setup continuous queries to fill the RPs
* backfill all data to all retention policies
* reduce period of `autogen` policy after backfilling all data

### Reduce HA data sent to InfluxDB

My main use case for the InfluxDB data is to graph primarily energy data over longer periods.
That is consumption and production of my smart meter, production and self-consumption of my solar power and energy usage of different appliances over the house, etc. Occasionally, or in the future, I might want to graph out temperature and humidity data of various sensors. I'm less interested to know in what state a certain switch or light was 4 months ago. 😑

As per the [docs](https://www.home-assistant.io/integrations/influxdb/#configure-filter) for the influxdb integration, we can achieve this by excluding entities, entity_globs or even whole domains. I ran some queries in my current influxdb to identify the most chatty and least useful sensors it contained and came to use below setup for now.

```yaml
influxdb:
  host: a0d7b954-influxdb
  port: 8086
  database: homeassistant
  username: !secret influxdb_username
  password: !secret influxdb_password
  max_retries: 3
  default_measurement: state
  exclude:
    entity_globs:
      - '*uptime*'
      - sensor.weather_*
      - update.*
      - camera.*
      - weather.*
      - script.*
    domains:
      - lock
      - select
      - switch
      - weather
      - zone
```

This will reduce about 80% of useless data being sent to influxdb.

I chose not to cleanup the existing data for these excluded sensors and domains for now, as the bulk of those will disappear when we apply a shorter retention period for our `autogen` RP later on. The current "useless" data will thus be downsampled into our new policies as well, but I'm OK with that for now.

### Setup new retention policies

With different retention policies, combined with continuous queries you can setup downsampling of incoming data with different aggregation levels, and control how long to keep that data. With this setup you can reduce your DEFAULT `autogen` retention policy.

I decided for the following setup:

| Retention Policy | Duration |
|---|---|
| autogen | 60d |
| rp_15min | 52w |
| rp_1h | ~~52w~~ infinite |
| ~~rp_1d~~ | ~~infinite~~ |

So keep full granularity of incoming data, keeping that for 60 days. That allows me to look closely at something in the full resolution for ~2 months after the facts, that should be plenty.
Downsampled data with 15 min resolution will be kept for 1 year. Initially I wanted to reduce that to 1 day resolution, but for now the 1h resolution will give me enough size reduction. So I set that to be kept indefinitely.

### Setup continuous queries

To downsample incoming data we use a continuous query. It will write into one the newly defined RP's. As I have created 2 new policies, I'll also need 2 CQ's to fill these when data comes in.


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

```

### Backfill data

Continuous queries only act on newly incoming data, and will not handle any data that's already present. If you have historical data in your default RP, you need to backfill the downsampled data to the other RPs.

I have about 14 months worth of data, but only the `rp_1h` data needs to be kept for > 52 weeks.

Because these queries will backfill all measurements at once, it takes quite some resources. To prevent the queries from timing out, I split them in several chunks. Make sure to run them one by one!

**rp_15min**

```sql
-- Backfill rp_15min for the past 52 weeks
SELECT mean(*) INTO "homeassistant"."rp_15min".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 52w and time < now() - 40w GROUP BY time(15m), * FILL(previous)
SELECT mean(*) INTO "homeassistant"."rp_15min".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 41w and time < now() - 30w GROUP BY time(15m), * FILL(previous)
SELECT mean(*) INTO "homeassistant"."rp_15min".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 31w and time < now() - 20w GROUP BY time(15m), * FILL(previous)
SELECT mean(*) INTO "homeassistant"."rp_15min".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 21w and time < now() - 10w GROUP BY time(15m), * FILL(previous)
SELECT mean(*) INTO "homeassistant"."rp_15min".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 11w and time < now() GROUP BY time(15m), * FILL(previous)

-- Backfill rp_1h, with all data
SELECT mean(*) INTO "homeassistant"."rp_1h".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 99w and time < now() - 50w GROUP BY time(1h), * FILL(previous)
SELECT mean(*) INTO "homeassistant"."rp_1h".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 52w and time < now() - 40w GROUP BY time(1h), * FILL(previous)
SELECT mean(*) INTO "homeassistant"."rp_1h".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 41w and time < now() - 30w GROUP BY time(1h), * FILL(previous)
SELECT mean(*) INTO "homeassistant"."rp_1h".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 31w and time < now() - 20w GROUP BY time(1h), * FILL(previous)
SELECT mean(*) INTO "homeassistant"."rp_1h".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 21w and time < now() - 10w GROUP BY time(1h), * FILL(previous)
SELECT mean(*) INTO "homeassistant"."rp_1h".:MEASUREMENT FROM "homeassistant"."autogen"./.*/ WHERE time > now() - 11w and time < now() GROUP BY time(1h), * FILL(previous)

```

### Verify downsampled data

After downsampling it's advised to perform some sanity checks on your data before cleaning the old data from the `autogen` RP. You can check some relevant fields via the InfluxDB addon, or adapt your Grafana dashboards to use the downsampled data for (some of) the graphs.

Once validated you can move to reducing the period of your default RP to flush old data out.

### Reduce period for DEFAULT RP

Now all data has been backfilled, and our continuous queries are handling downsampling of incoming data, it is safe to update the retention policy on your `autogen` policy.
I've updated it to 60 days, so all older data will get cleaned.

## Results

After the backfill, the data store looks like this:

```bash
# pwd
/mnt/data/supervisor/addons/data/a0d7b954_influxdb/influxdb/data/homeassistant
# du -h -d1
32.0M	./_series
44.8M	./rp_15min
2.3G	./autogen
20.6M	./rp_1h
2.4G	.

```

So our default `autogen` RP went from 17.6G to 2.3G! That's an 87% reduction in storage usage + the benefit of way faster Grafana dashboards.

{{< img name="rp-overview" lazy=false >}}

Now to see if the reduced total size of the HA install leads to less issues with my backups.