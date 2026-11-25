# TODO

## Publish to Home Assistant via MQTT

Optional; the Prometheus path must never depend on it.

- **Configuration**
  - Off by default.
  - Enabled with a new CLI option for the broker (`host[:port]`), passed through `prometheus_temperatured.defaults` like the other options.
  - Credentials come from an environment variable or a file, never from argv.
- **MQTT discovery**, so entities appear in HA automatically:
  - A config topic per location and measurement, e.g. `homeassistant/sensor/<sensor_name>_temperature/config` and `..._humidity/config`. Generate these from the `MEASUREMENTS` table, so CO2 etc. follow automatically.
  - `device_class` `temperature`/`humidity`/`carbon_dioxide`, `unit_of_measurement` `°C`/`%`/`ppm`, `state_class: measurement`.
  - `unique_id` derived from `sensor_name`. That keys entities on the name, as Prometheus does.
  - Group entities as one HA device **per location** (`name`). A location can have several sensors with different `model`/`fw`/`version`, so don't take the device's model from a single packet. Use a neutral device model (e.g. "pico-temperature location") and expose the per-sensor details as entity attributes, or from `sensor_info`.
  - `expire_after` about 3–6× the 10 s send interval, so a silent location shows as unavailable.
  - Publish config messages retained, once per location (when first seen and after a reconnect), not with every packet.
- **Publish only accepted measurements.** Values rejected by validation or the ranges file never reach HA.
- **Never let MQTT affect Prometheus**: if the broker is down, slow or misconfigured, log it and keep going.
  - Use the client's background loop and reconnect handling.
  - Never block packet handling on a publish.
- Dew point and absolute humidity stay out of the daemon. HA calculates its own, and Prometheus uses `prometheus_temperatured.rules.yml`.

## Repo

- Check that the sensor repo's template `config.py` now defaults `UDP_PORT` to `11235`, so the README's "the defaults match" is true.

## Ideas

- **Stale gauges:** when a probe dies, its gauges keep exporting the last value, so graphs show a flat line rather than a gap. One option is to remove a probe's measurement series after a timeout. The last-seen alerts already cover this, so it's cosmetic.
- **Reload the ranges file on SIGHUP**, rather than needing a restart.
