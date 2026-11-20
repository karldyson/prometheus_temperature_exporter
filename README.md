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
- **Temperature and humidity**: humidity is exported for probes that send it (e.g. SHT30), ready for more measurement types
- **Health metrics**: spot a probe that stops sending, and a sensor on a probe that stops giving good values
- **Per-sensor ranges**: discard implausible values (e.g. a DS18B20's 85.0 power-on value) with a range per sensor
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
- **-r \<file>**: A per-sensor measurement ranges file. Defaults to none, using the built-in ranges.

### Measurement Ranges

Each measurement is checked before it's used. A value that isn't a number (including `null`), or is
outside its range, is discarded, logged, and counted in `sensor_measurement_rejected_total`. The other
measurements in the same packet are still used.

Ranges come from the file given with `-r` (see the template `prometheus_temperatured.ranges`):

```ini
[DEFAULT]
temperature = -10:45
humidity = 0:100

[BoilerFlow]
temperature = -10:110
```

- `measurement = min:max`, inclusive.
- `[DEFAULT]` applies to every sensor.
- A section named after a sensor (its `sensor_name`, i.e. the name with spaces removed) overrides only the measurements it lists.
- Anything not set in the file uses the built-in range: temperature `-55:125` (the DS18B20's spec), humidity `0:100`.
- The file is read at startup; restart the daemon after changing it. A mistake in the file stops the daemon from starting, with a message saying what's wrong.

The built-in temperature range is deliberately wide, so on its own it won't catch the DS18B20's error
values (`85.0` at power-on, `-127` when disconnected). A sane `[DEFAULT]` for rooms, with overrides for
the heating system, loft, outside etc., will.

## Metrics

| Metric | Labels | Meaning |
|---|---|---|
| `temperature` | `friendly_name`, `sensor_name` | Temperature in Celsius |
| `humidity` | `friendly_name`, `sensor_name` | Relative humidity in %; only for probes that send it |
| `sensor_last_seen_timestamp` | `friendly_name`, `sensor_name` | When we last received a valid packet from the probe, whatever its measurements were. Is the probe alive? |
| `sensor_measurement_last_seen_timestamp` | `friendly_name`, `sensor_name`, `measurement` | When we last accepted that measurement from the probe. Is that sensor on the probe still giving good values? |
| `sensor_measurement_rejected_total` | `sensor_name`, `measurement`, `reason` | Count of discarded measurements; `reason` is `invalid` or `out_of_range` |
| `sensor_info` | `sensor_name`, `model`, `fw`, `version` | Always `1`; the probe's details as labels. Fields an older probe doesn't send are empty. When they change, the old label set is removed. |

The firmware version is deliberately a label on `sensor_info` and not on the measurement metrics, so a
firmware update doesn't start a new series for every graph.

**Upgrading from 1.x:** `temperature_last_seen_timestamp` is replaced by `sensor_last_seen_timestamp`
and `sensor_measurement_last_seen_timestamp`, and 1.1's `sensor_version` label is gone. Update queries
before upgrading; see [Queries](#queries).

## Packets

Each probe multicasts one JSON packet per reading, e.g. a newer probe with an SHT30:

```json
{"type": "reading", "model": "sht30", "name": "MainBedroom", "sensor": "1a2b3c4d", "temperature": 20.92, "humidity": 49.9, "offset": 0.0, "fw": "1.24.1", "version": "1.1.0"}
```

and an older probe with a DS18B20:

```json
{"type": "temperature", "name": "ColdWaterTank", "sensor": "28ff...", "temperature": 4.5, "offset": 0.0, "version": "1.0"}
```

| Field | Required | Meaning |
|---|---|---|
| `type` | yes | `"reading"`, or `"temperature"` from older probes (still accepted). Packets of any other type are ignored. |
| `name` | yes | The probe's name, becomes `sensor_name` (spaces removed) and `friendly_name` |
| `temperature` | no | Celsius |
| `humidity` | no | Relative humidity, % |
| `model` | no | Sensor chip, e.g. `ds18b20`, `sht30` |
| `fw` | no | MicroPython release on the probe |
| `version` | no | Probe code version |
| `sensor`, `offset` | no | Informational, not exported |

A packet should carry at least one measurement. If a sensor read fails, the probe should leave the field
out or send `null`, never a made-up value. Either way, that measurement's
`sensor_measurement_last_seen_timestamp` stops advancing, which is what the alerts below look for.

**Deploy the daemon (2.0 or later) before any probe that sends `"type": "reading"`**: 1.x ignores those packets.

### Glossary

- **probe**: the Pico device. Its `name` becomes the `sensor_name` label (which keeps that name so existing graphs carry on working).
- **sensor**: a chip on the probe (`model`), e.g. DS18B20, SHT30.
- **measurement**: one quantity's value, e.g. `temperature`, `humidity`; also the `measurement` label.
- **reading**: one packet; a probe's measurements at one moment (`"type": "reading"`).

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

The same works for `humidity`.

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

- A probe has stopped sending:
  `time() - max by (sensor_name) (max_over_time(sensor_last_seen_timestamp[1d])) > 60`
- A sensor on a probe has stopped giving good values:
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
