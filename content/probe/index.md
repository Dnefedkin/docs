---
title: "Probe"
date: 2017-07-25T17:24:32-07:00
weight: 20
---
Cloudprober's main task is to run probes. A probe executes something, usually
against a set of targets, to verify that the systems are working as expected.
For example, an HTTP probe executes an HTTP request against a web server to
verify that the web server is available. Cloudprober probes run repeatedly at a
configured interval and export probe results as a set of metrics.

A probe is defined as a set of the following fields:

 Field   | Description
---------|---------
type     | Probe type, for example: HTTP, PING or UDP
name     | Probe name. Each probe should have a unique name.
interval | How often to run the probe (in milliseconds).
timeout  | Probe timeout (in milliseconds).
targets  | Targets to run probe against.

Each probe also has optional probe-type specific config, for example,
relative_url for HTTP probes. All probe types, except for the external probe
type (explained below), export following metrics at a minimum:

|Metric | Description|
|-------|------------|
|sent   | Total number of requests sent. Type of request depends on the probe <br> type (ICMP request for PING, HTTP request for HTTP, etc). |
|rcvd   | Total number of responses received.|
|rtt    | Total roundtrip time for the responses received (in microseconds).|


## Probe Types

Cloudprober has built-in support for the following probe types:

* [Ping](#ping)
* [HTTP](#http)
* [UDP](#udp)
* [DNS](#dns)
* [External](#external)

### Ping

[`Code`](http://github.com/google/cloudprober/tree/master/probes/ping) | [`Config
options`](http://github.com/google/cloudprober/tree/master/probes/ping/config.proto)

Ping probe type implements a fast ping prober, that can probe hundreds of
targets in parallel. Probe results are reported as number of packets sent,
received and round-trip time. It supports raw sockets (requires root access) as
well as datagram sockets for ICMP (doesn't require root access).

ICMP datagram sockets are not enabled by default on most Linux systems. You can
enable them by running the following command:
`sudo sysctl -w net.ipv4.ping_group_range="0 5000"`

### HTTP

[`Code`](http://github.com/google/cloudprober/tree/master/probes/http) | [`Config
options`](http://github.com/google/cloudprober/tree/master/probes/http/config.proto)

HTTP probe is be used to send HTTP(s) requests to a target and verify that a
reponse is received. Probe results are reported as number of requests sent,
responses received, overall round-trip time and a map of response code counts.
Requests are marked as failed (not counted towards received) if there is a
timeout.

### UDP

[`Code`](http://github.com/google/cloudprober/tree/master/probes/udp) | [`Config
options`](http://github.com/google/cloudprober/tree/master/probes/udp/config.proto)

UDP probe type sends a UDP packet to the configured targets. UDP probe (and all
other probes that use ports) provides more coverage for the network elements on
the data path as most forwarders use 5-tuple hashing and usign a new source port
for each probe ensures that we hit different network element each time.

### DNS

[`Code`](http://github.com/google/cloudprober/tree/master/probes/dns) | [`Config
options`](http://github.com/google/cloudprober/tree/master/probes/dns/config.proto)

DNS probe type is implemented in a similar way as other probes except for that
it sends DNS requests to the target.

### External

[`Code`](http://github.com/google/cloudprober/tree/master/probes/external) | [`Config
options`](http://github.com/google/cloudprober/tree/master/probes/external/config.proto)

External probe type allows running arbitrary probes through cloudprober. For an
external probe, actual probe logic resides in an external program; cloudprober
only manages the execution of that program and provides a way to export that
data through the standard channel.

External probe can be configured in two modes:

*  __ONCE__:
   In this mode, external program is executed again for probe run and results
   are returned on stdout. This is a simple model but it doesn't allow the
   external program to maintain state and multiple forks can be expensive
   depending on the frequency of the probes.

*  __SERVER__:
   In this mode, external program is executed only once (or whenever it stops
   running) and is expected to run in a daemon mode. Cloudprober and external
   probe process communicate with each other over stdin/stdout using RPC.
