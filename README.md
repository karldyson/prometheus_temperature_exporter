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
1. Copy `prometheus_temperatured.service` to `/etc/systemd/system/`
1. Copy `prometheus_temperatured.defaults` to `/etc/default/prometheus_temperatured`
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

## Grafana

I'm kinda new to this, for sure, so I'm very much open to you correcting me or making
suggestions.

### Friendly Names

Each sensor has a sensor name in the config on the pico, which gets chucked in the multicast packet.

So, for example. I have one called ColdWaterTank.

To make these names more friendly for Grafana output, I split them on each capital letter and output them with spaces, so the above becomes "Cold Water Tank".

### Dashboard

I've included an example dashboard but you will likely want to change it.

I have a main graph that dynamically picks up all sensors. There's a picker variable
at the top of the dashboard so you can look at subsets.

I then have some hard coded grouped graphs, for example, one shows the sensors
related to my heating system.

### ALerts

I have a basic alert set up to notify me if the temperature of the water in the cold tank in the loft goes below 3 celcius.

## TODO

## Copyright

Copyright (c) 2025 Karl Dyson

## License

Licensed under the Apache License, Version 2.0. See the LICENSE file for details.

## Warranty

There's no warranty either implicit or implied that suggests this code is fit for purpose.

I am not responsible if you use this code without checking its suitability for your use
case and wake up one morning to find your cat is too cold, or too warm.
