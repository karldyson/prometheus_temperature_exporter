# Prometheus Temperature Exporter

This is designed to work in conjunction with the Pico Temperature Monitor.

That multicasts the temperatures onto the network so that anything that wishes to
consume temperatures can just subscribe to the multicast group.

This daemon receives those multicast packets and updates sensors such that Prometheus
can scrape them.

You know the rest of the drill. You can then draw graphs in Graphana, alert if the cat
basket is too cold, etc.

## Features

- **Automatic addition of new sensors**: New sensors are automatically detected and added
- **Temperature, humidity and CO2**: each is exported for the locations that send it (e.g. humidity from an SHT30, CO2 from an SCD41)
- **Locations**: several sensors can report for one location (e.g. an SHT30 and an SCD41 both named `Kitchen`)
- **Health metrics**: spot a location that stops sending, and a sensor that stops giving good values
- **Per-location ranges**: discard implausible values (e.g. a DS18B20's 85.0 power-on value) with a range per sensor
- **Robust**: a malformed packet is logged and skipped rather than stopping the daemon

## Installation

**Note:** You may want to check that your network supports multicast, doesn't block it (intentionally or otherwise).

1. Copy `prometheus_temperatured` to `/usr/local/bin/`
1. Copy `prometheus_temperatured.service` to `/etc/systemd/system/`
1. Copy `prometheus_temperatured.defaults` to `/etc/default/prometheus_temperatured`
1. Copy `prometheus_temperatured.ranges` to `/etc/prometheus_temperatured.ranges`, edit it to suit your sensors, and add `-r /etc/prometheus_temperatured.ranges` to `TEMPERATURED_OPTIONS` in `/etc/default/prometheus_temperatured` (optional, but recommended; see [Measurement Ranges](#measurement-ranges))
1. Enable the service: `sudo systemctl enable prometheus_temperatured.service`
1. Start the service: `sudo systemctl start prometheus_temperatured.service`

Your probes should be automatically picked up.

6. Configure Prometheus

   ...add the following snippet to your `prometheus.yml` configuration.

   ```
   - job_name: 'temperature'
     static_configs:
       - targets: ['192.0.2.100:8000']
   ```

7. Check the config `sudo promtool check config /path/to/prometheus.yml`
8. HUP Prometheus to pick up the config change `killall -HUP prometheus`

## Configuration

You will want to check that the default config options work for your setup.

The default here match with the defaults in the code running on the pico if you stuck to them.

### Options

- **-s \<integer>**: The TCP port that the HTTP server will listen on for connections from prometheus for the purposes of scraping the temperature metrics. Defaults to `8000`.
- **-i \<string>**: The IP address that the multicast listener should bind to. Defaults to `0.0.0.0`.
- **-g \<string>**: The multicast group that the multicast packets are being sent to. Defaults to `239.0.0.1`.
- **-p \<integer>**: The UDP port that the multicast packets are expected to arrive on. Defaults to `11235`.
- **-r \<file>**: A per-location measurement ranges file. Defaults to none, using the built-in ranges.

### Measurement Ranges

Each measurement is checked before it's used. A value that isn't a number (including `null`), or is
outside its range, is discarded, logged, and counted in `sensor_measurement_rejected_total`. The other
measurements in the same packet are still used.

Ranges come from the file given with `-r` (see the template `prometheus_temperatured.ranges`):

```ini
[DEFAULT]
temperature = -10:45
humidity = 0:100
co2 = 300:10000

[BoilerFlow]
temperature = -10:110
```

- `measurement = min:max`, inclusive.
- `[DEFAULT]` applies to every location.
- A section named after a location (its `sensor_name`, i.e. the name with spaces removed) overrides only the measurements it lists. It applies to every sensor reporting for that location.
- Anything not set in the file uses the built-in range: temperature `-55:125` (the DS18B20's spec), humidity `0:100`, co2 `0:40000` (the SCD4x's output range).
- The file is read at startup; restart the daemon after changing it. A mistake in the file stops the daemon from starting, with a message saying what's wrong.

The built-in temperature range is deliberately wide, so on its own it won't catch the DS18B20's error
values (`85.0` at power-on, `-127` when disconnected). A sane `[DEFAULT]` for rooms, with overrides for
the heating system, loft, outside etc., will.

## Metrics

| Metric | Labels | Meaning |
|---|---|---|
| `temperature` | `friendly_name`, `sensor_name` | Temperature in Celsius |
| `humidity` | `friendly_name`, `sensor_name` | Relative humidity in %; only for locations that send it |
| `co2` | `friendly_name`, `sensor_name` | CO2 concentration in ppm; only for locations that send it |
| `sensor_last_seen_timestamp` | `friendly_name`, `sensor_name` | When we last received a valid packet for the location, whatever its measurements were. With several sensors at one location, this only shows that at least one of them is still sending. |
| `sensor_measurement_last_seen_timestamp` | `friendly_name`, `sensor_name`, `measurement` | When we last accepted that measurement for the location. Each measurement at a location comes from one sensor, so this is what catches a single dead sensor in a shared location. |
| `sensor_measurement_rejected_total` | `sensor_name`, `measurement`, `reason` | Count of discarded measurements; `reason` is `invalid` or `out_of_range` |
| `sensor_info` | `sensor_name`, `sensor_id`, `model`, `fw`, `version` | Always `1`; one series per sensor at each location, with the sensor's details as labels. `sensor_id` is the packet's `sensor` field. Fields an older probe doesn't send are empty. When they change, that sensor's old label set is removed. |

The `sensor_name` label carries the **location** (the label name is kept so existing graphs carry on working).

The firmware version is deliberately a label on `sensor_info` and not on the measurement metrics, so a
firmware update doesn't start a new series for every graph.

**Upgrading from 2.x:** `sensor_info` has a new `sensor_id` label, so its series change. A query that
joins on `sensor_info` by `sensor_name` alone may now match several series for one location; add
`sensor_id` to the join, or aggregate first. The new `co2` metric needs no changes elsewhere.

**Upgrading from 1.x:** `temperature_last_seen_timestamp` is replaced by `sensor_last_seen_timestamp`
and `sensor_measurement_last_seen_timestamp`, and 1.1's `sensor_version` label is gone. Update queries
before upgrading; see [Queries](#queries).

## Packets

Each unit multicasts one JSON packet per reading, one per sensor, e.g. a newer unit with an SHT30:

```json
{"type": "reading", "model": "sht30", "name": "MainBedroom", "sensor": "1a2b3c4d", "temperature": 20.92, "humidity": 49.9, "offset": 0.0, "fw": "1.24.1", "version": "1.1.0"}
```

an SCD41 at the same location, which by default sends only CO2:

```json
{"type": "reading", "model": "scd41", "name": "MainBedroom", "sensor": "1cab65073b57", "co2": 466, "fw": "1.24.1", "version": "3.0.0"}
```

and an older probe with a DS18B20:

```json
{"type": "temperature", "name": "ColdWaterTank", "sensor": "28ff...", "temperature": 4.5, "offset": 0.0, "version": "1.0"}
```

| Field | Required | Meaning |
|---|---|---|
| `type` | yes | `"reading"`, or `"temperature"` from older probes (still accepted). Packets of any other type are ignored. |
| `name` | yes | The location, becomes `sensor_name` (spaces removed) and `friendly_name` |
| `temperature` | no | Celsius |
| `humidity` | no | Relative humidity, % |
| `co2` | no | CO2, ppm |
| `sensor` | no | The sensor's ID (DS18B20 ROM code or chip serial), exported as `sensor_id` on `sensor_info` |
| `model` | no | Sensor chip, e.g. `ds18b20`, `sht30`, `scd41` |
| `fw` | no | MicroPython release on the probe |
| `version` | no | Probe code version |
| `offset`, `rh_offset` | no | Informational, not exported |

No measurement is present in every packet: from firmware 3.0.0, `temperature` is only there if that sensor publishes it. A packet should carry at least one measurement. If a sensor read fails, the probe should leave the field
out or send `null`, never a made-up value. Either way, that measurement's
`sensor_measurement_last_seen_timestamp` stops advancing, which is what the alerts below look for.

**Deploy the daemon (2.0 or later) before any probe that sends `"type": "reading"`**: 1.x ignores those packets.

### Glossary

- **unit**: one Pico (or other board) running the firmware. A unit can have several sensors attached.
- **sensor**: one physical sensor chip or probe attached to a unit (a DS18B20, SHT30, SCD41...). It's identified in packets by `sensor` (the DS18B20 ROM code or the chip serial) and `model`.
- **location**: what a reading is *about*, set by `name` (e.g. `Kitchen`, `BoilerFlow`). Several sensors can report for the same location.
- **measurement**: one quantity: `temperature`, `humidity`, `co2`.
- **reading**: one packet. One sensor's published measurements for its location at one moment.

The `sensor_name` label carries the **location**, not the sensor (the label name is kept so existing graphs carry on working). The `measurement` label holds the measurement's name.

## Grafana

I'm kinda new to this, for sure, so I'm very much open to you correcting me or making
suggestions.

### Friendly Names

Each sensor has a sensor name in the config on the pico, which gets chucked in the multicast packet.

So, for example. I have one called ColdWaterTank.

To make these names more friendly for Grafana output, I split them on each capital letter and output them with spaces, so the above becomes "Cold Water Tank".

### Queries

Aggregate by `friendly_name, sensor_name` so that series from older versions (1.1 had a
`sensor_version` label) join up into one line per sensor:

```
avg_over_time(max by (friendly_name, sensor_name) (temperature{friendly_name=~"$friendly_name"})[1m:])
```

The same works for `humidity` and `co2`.

### Dew Point and Absolute Humidity

The daemon only exports the raw `temperature` and `humidity`. `prometheus_temperatured.rules.yml`
contains Prometheus recording rules that derive, for every probe reporting both:

- `sensor:dewpoint`: dew point in Celsius (Magnus formula)
- `sensor:absolute_humidity`: grams of water per cubic metre

Installation instructions are at the top of the file. They only produce values from when they're loaded.

### Dashboard

I've included an example dashboard but you will likely want to change it.

I have a main graph that dynamically picks up all sensors. There's a picker variable
at the top of the dashboard so you can look at subsets.

I then have some hard coded grouped graphs, for example, one shows the sensors
related to my heating system.

### Alerts

I have a basic alert set up to notify me if the temperature of the water in the cold tank in the loft goes below 3 celcius.

Some suggested health alerts:

- A location has stopped sending:
  `time() - max by (sensor_name) (max_over_time(sensor_last_seen_timestamp[1d])) > 60`
- A sensor has stopped giving good values (use this one for locations with several sensors, where the alert above only fires once all of them stop):
  `time() - max by (sensor_name, measurement) (max_over_time(sensor_measurement_last_seen_timestamp[1d])) > 60`
- A sensor is flaky:
  `sum by (sensor_name, measurement) (increase(sensor_measurement_rejected_total[15m])) > 0`

The `max_over_time(...[1d])` keeps the first two alerting if the daemon restarts while something is
already broken (the daemon won't recreate a series it hasn't heard from since it started).

## TODO

See [TODO.md](TODO.md).

## Copyright

Copyright (c) 2025 Karl Dyson

## License

Licensed under the Apache License, Version 2.0. See the LICENSE file for details.

## Warranty

There's no warranty either implicit or implied that suggests this code is fit for purpose.

I am not responsible if you use this code without checking its suitability for your use
case and wake up one morning to find your cat is too cold, or too warm.
