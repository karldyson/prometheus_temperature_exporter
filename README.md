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

## Installation

**Note:** You may want to check that your network supports multicast, doesn't block it (intentionally or otherwise).

1. Copy `prometheus_temperatured` to `/usr/local/bin/`
2. Copy `prometheus_temperature.service` to `/etc/systemd/system/`
3. Enable the service: `sudo systemctl enable prometheus_temperature.service`
4. Start the service: `sudo systemctl start prometheus_temperature.service`

Your probes should be automatically picked up.

5. Configure Prometheus

   ...add the following snippet to your `prometheus.yml` configuration.

   ```
   - job_name: 'temperature'
     static_configs:
       - targets: ['192.0.2.100:8000']
   ```

6. Check the config `sudo promtool check config /path/to/prometheus.yml`
7. HUP Prometheus to pick up the config change `killall -HUP prometheus`

## Grafana

I'm kinda new to this, for sure, so I'm very much open to you correcting me or making
suggestions.

### Dashboard

I've included an example dashboard but you will likely want to change it.

I have a main graph that dynamically picks up all sensors. There's a picker variable
at the top of the dashboard so you can look at subsets.

I then have some hard coded grouped graphs, for example, one shows the sensors
related to my heating system.

### ALerts

I have a basic alert set up to notify me if the temperature of the water in the cold tank in the loft goes below 3 celcius.

## TODO

1. Pull multicast group, port, listen-on IP, etc out into configuration
1. Send a timestamp for every sensor update?

## Copyright

Copyright (c) 2025 Karl Dyson

## License

Licensed under the Apache License, Version 2.0. See the LICENSE file for details.

## Warranty

There's no warranty either implicit or implied that suggests this code is fit for purpose.

I am not responsible if you use this code without checking its suitability for your use
case and wake up one morning to find your cat is too cold, or too warm.
