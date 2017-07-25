---
title: Getting started
weight: 10
---

## Installation

Getting started with Cloudprober is as easy as running the following command:

```
docker run --net host -v /tmp:/tmp cloudprober/cloudprober
# Note: --net host provides better network performance and makes port forwarding
# management easier.
```

This will start cloudprober with only the "sysvars" module (no probes). It will
write metrics to the stdout in cloudprober's line protocol format (to be
documented). It will also start a Prometheus exporter that can be accessed at:
http://localhost:9313

If docker is not an option, you can also download the pre-built binaries
directly from the Github's
[releases](http://github.com/google/cloudprober/releases) page.

Since sysvars variables are not very interesting themselves, lets add a simple
config that probes Google's homepage:

```shell
# Write config to a file in /tmp
cat > /tmp/cloudprober.cfg <<EOF
probe {
  name: "google_homepage"
  type: HTTP
  targets {
    host_names: "www.google.com"
  }
  interval_msec: 5000  # 5s
  timeout_msec: 1000   # 1s
}
EOF
```

You can have cloudprober use this config file using one of the following
methods:

```shell
docker run --net host -v /tmp/cloudprober.cfg:/etc/cloudprober.cfg \
    -v /tmp:/tmp cloudprober/cloudprober

# Non-docker
./cloudprober --config_file /tmp/cloudprober.cfg
```

You'll see probe metrics at the URL: http://hostname:9313/metrics and at the
stdout:

```
cloudprober 1500590430132947313 1500590520 labels=ptype=http,probe=google-http,dst=www.google.com sent=17 rcvd=17 rtt=1808357 timeouts=0 resp-code=map:code,200:17
cloudprober 1500590430132947314 1500590530 labels=ptype=sysvars,probe=sysvars hostname="manugarg-workstation" uptime=100
cloudprober 1500590430132947315 1500590530 labels=ptype=http,probe=google-http,dst=www.google.com sent=19 rcvd=19 rtt=2116441 timeouts=0 resp-code=map:code,200:19
```

Since this is not very interesting by itself, let's run a prometheus instance to
scrape our metrics and generate pretty graphs.

## Running Prometheus

Download prometheus binary from the prometheus [release
page](https://prometheus.io/download/). You can use a config like the following
to scrape cloudprober running on the local host.

```shell
# Write config to a file in /tmp
cat > /tmp/prometheus.yml <<EOF
scrape_configs:
  - job_name: 'cloudprober'
    scrape_interval: 10s
    static_configs:
      - targets: ['localhost:9313']
EOF

Start prometheus:
./prometheus --config.file=/tmp/prometheus.yml
```

Prometheus provides a web interface at http://localhost:9090. You can explore
the probe metrics and build useful graphs through this interface. All core
probes in cloudprober export at least 3 counters:

*   _sent_: Number of requests sent (type of request depends on the probe type).
*   _rcvd_: Number of responses received.
*   _rtt_:  Total round trip time in microseconds.

Using these counters, loss and latency can be calculated as:

```
loss    = (rate(sent) - rate(rcvd)) / rate(sent)
latency = rate(rtt) / rate(rcvd)
```

Assuming that prometheus is running at `localhost:9090`, graphs depicting loss
ratio and latency over time can be accessed in prometheus at: [loss and
latency](http://localhost:9090/graph?g0.range_input=1h&g0.expr=\(rate\(sent%5B1m%5D\)+-+rate\(rcvd%5B1m%5D\)\)+%2F+rate\(sent%5B1m%5D\)&g0.tab=0&g1.range_input=1h&g1.expr=rate\(rtt%5B1m%5D\)+%2F+rate\(rcvd%5B1m%5D\)+%2F+1000&g1.tab=0).
