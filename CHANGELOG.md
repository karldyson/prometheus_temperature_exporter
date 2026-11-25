# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project
uses version numbers in the style of [Semantic Versioning](https://semver.org/): a major version
bump means a breaking change to the exported metrics, the CLI or the packets accepted.

## [3.0] - 2026-11-25

Breaking change to `sensor_info`. Packets from every existing probe are still accepted unchanged.
See "Upgrading from 2.x" in the README.

### ⚠️ Breaking
- `sensor_info` has a new `sensor_id` label, from the packet's `sensor` field. There is now one series
  per sensor at each location instead of one per location.

### Added
- A `co2` gauge (ppm) for sensors that send it (e.g. SCD41, from pico-temperature 3.0.0), with a
  built-in range of `0:40000`. It can be set in the ranges file like the other measurements.
- README glossary of unit, sensor, location, measurement and reading, shared with pico-temperature.
  A `name` is a location, and several sensors can report for one location.

### Changed
- Docs: `sensor_last_seen_timestamp` is per location, so in a shared location only
  `sensor_measurement_last_seen_timestamp` catches a single dead sensor. Ranges apply per location.

### Fixed
- `sensor_info` no longer flaps (removing and re-adding its label set with every packet) when
  several sensors with different models share a location.

## [2.0] - 2026-11-20

Breaking changes to the exported metrics. Packets from existing probes are still accepted unchanged.
See "Upgrading from 1.x" in the README.

### Added
- Humidity support: a `humidity` gauge (relative humidity, %) for probes that send it, e.g. SHT30.
- A new packet type, `"type": "reading"`, carrying any mix of measurements, so a probe doesn't have
  to report temperature. The legacy `"type": "temperature"` is still accepted.
- `sensor_last_seen_timestamp`: updated for every valid packet. Shows whether the probe is alive.
- `sensor_measurement_last_seen_timestamp{measurement}`: updated when a measurement is accepted.
  Shows whether a sensor on the probe is still giving good values.
- `sensor_measurement_rejected_total{reason}`: counts discarded measurements (`invalid`, `out_of_range`).
- `sensor_info{model, fw, version}`: the probe's details as labels on a metric with the value 1.
  When they change, the old label set is removed.
- A per-sensor ranges file (`-r`): measurements outside a sensor's range are discarded. `[DEFAULT]`
  applies to every sensor, and a template is included. Without the file, the daemon falls back to
  built-in ranges.
- `prometheus_temperatured.rules.yml`: Prometheus recording rules for `sensor:dewpoint` and
  `sensor:absolute_humidity`.
- README sections on metrics, packet format, glossary, ranges, queries and alerts.

### Changed
- Each measurement in a packet is validated on its own, so one bad value no longer discards the others.
- Optional packet fields (`model`, `fw`, `version`) can be missing.

### Removed
- `temperature_last_seen_timestamp`, replaced by `sensor_last_seen_timestamp` and
  `sensor_measurement_last_seen_timestamp`.
- The `sensor_version` label added in 1.1. Firmware updates no longer start a new series.

### Fixed
- Malformed packets, and packets without a `version`, are logged and skipped instead of stopping the daemon.
- The previous label set is no longer exported, frozen at its last value, after a probe's version changes.

## [1.1] - 2026-11-20

### Added
- `sensor_version` label on `temperature` and `temperature_last_seen_timestamp`, taken from the
  probe's `version` field.

### Known issues
- A probe firmware update starts a new series, and the previous label set keeps being exported,
  frozen at its last value, until the daemon restarts.
- Packets without a `version` field (older probes) raise an error and stop the daemon.
- A malformed packet (bad JSON, missing field) stops the daemon. This also affects 1.0.

## [1.0] - 2025-12-16

Initial release.

### Added
- Listens for JSON temperature packets multicast by the Pico probes and exports them for
  Prometheus to scrape.
- `temperature` gauge, labelled with `sensor_name` (the probe name with spaces removed) and
  `friendly_name` (CamelCase split into words, e.g. `ColdWaterTank` → "Cold Water Tank").
- `temperature_last_seen_timestamp` gauge, so you can alert when a probe stops reporting.
- New probes are picked up automatically.
- Command line options: `-s` HTTP port, `-i` bind address, `-g` multicast group,
  `-p` multicast port, `-h` help.
- systemd service file, plus a defaults file for passing options to the service.

[3.0]: https://github.com/karldyson/prometheus_temperature_exporter/compare/v2.0...v3.0
[2.0]: https://github.com/karldyson/prometheus_temperature_exporter/compare/v1.1...v2.0
[1.1]: https://github.com/karldyson/prometheus_temperature_exporter/compare/9d86a46...v1.1
[1.0]: https://github.com/karldyson/prometheus_temperature_exporter/commit/9d86a46
