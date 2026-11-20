# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project
uses version numbers in the style of [Semantic Versioning](https://semver.org/): a major version
bump means a breaking change to the exported metrics, the CLI or the packets accepted.

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

[1.1]: https://github.com/karldyson/prometheus_temperature_exporter/compare/9d86a46...69067ef
[1.0]: https://github.com/karldyson/prometheus_temperature_exporter/commit/9d86a46
