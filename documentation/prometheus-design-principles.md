# Prometheus design principles

What is prometheus for?  There seem to be a number of design
principles behind prometheus that aren't explicitly documented on the
website.  However, prometheus maintainers sometimes refer to them in
mailing lists or github issues.  This page tries to reverse engineer
them.

## Terminology

The Prometheus community uses certain words as jargon terms.  These
are some:

  * failure domain: a group of things that can all fail at once (a
    data centre, an AWS region or AZ)
  * instance: an endpoint that you can scrape
  * job: a collection of *instances* with the same purpose, replicated
    for scalability or reliability or some other purpose

## What Prometheus does

Prometheus is a tool for collecting time-series data.  These
time-series can be queried using PromQL, a custom query language for
time-series data. PromQL queries can be used as a basis for monitoring
systems through dashboards and alerting.

## What Prometheus doesn't do

[Julius Volz listed some things Prometheus doesn't do](https://youtu.be/QgJbxCWRZ1s?t=9m28s):

  * raw log / event collection (use a logging system)
  * request tracing (use zipkin or similar)
  * "magic" anomaly detection
  * durable long-term storage (use an external system for this,
    decouple it from your operational monitoring)
  * automatic horizontal scaling
  * user / auth management

## Pull model

Prometheus is a pull-based system.
[Julius Volz listed some benefits of pull](https://youtu.be/QgJbxCWRZ1s?t=17m55s):

  * automatic upness monitoring - if you fail to scrape metrics, you
    can immediately say something is wrong
  * "horizontal monitoring" - Team A can include some metrics from
    Team B on their operational dashboard by selectively scraping.
    You can be flexible with what you pull from where
  * run a copy of monitoring on your laptop, scraping the same endpoints
      * experiment with different alerting rules
  * simpler HA - just deploy a copy (alertmanager deduplicates alerts)
  * less configuration - instances don't need to know where monitoring
    lives
  * the monitoring system is in control of when it ingests new metrics
    by setting a scraping interval.  This avoids issues where a
    misbehaving client emits so many metrics they negatively impact
    other users of the monitoring system.

The
[prometheus docs also say](https://prometheus.io/docs/introduction/faq/#why-do-you-pull-rather-than-push?):

> Overall, we believe that pulling is slightly better than pushing,
> but it should not be considered a major point when considering a
> monitoring system.

### When to use pushgateway

There's a
[page in the docs for when to use pushgateway](https://prometheus.io/docs/practices/pushing/).

## Shape of data

### Names

  * labels should not have too high a cardinality
      * See the CAUTION here: https://prometheus.io/docs/practices/naming/#labels
      * it seems that prometheus 2.0's new storage engine took a lot
      of effort to support "churn" of labels, so long as the number of
      active labels at any point is relatively small. see
      ["storing 16 bytes at scale" from promcon 2017](https://promcon.io/2017-munich/talks/storing-16-bytes-at-scale/)
      or
      [prometheus 2.0: new storage layer...](https://coreos.com/blog/prometheus-2.0-storage-layer-optimization)
  * labels should not include redundant informational data

### Values

  * [counters should always count up from zero](https://www.robustperception.io/how-does-a-prometheus-counter-work/)
      * data should always make sense, independent of scrape interval
      * this is in contrast to statsd's behaviour
  * [sum of rates, not rate of sums](https://www.robustperception.io/rate-then-sum-never-sum-then-rate/)

## Architectural things ##

### Simplicity ###

Prometheus is designed to achieve reliability through simplicity.

This means:
  * redundancy is achieved by deploying multiple separate instances,
    not by database replication.
  * recovery from backup is not a priority.  if a prometheus server
    has a disk failure, just use another redundant instance (assuming
    you have one)
  * if a prometheus instance has a temporary blip (say, a network
    issue) and then recovers, it will have missing metrics for the
    duration of the issue.
    
[Julius Volz talked about why they didn't do clustering](https://youtu.be/QgJbxCWRZ1s?t=26m40s).

### Prometheus should be deployed in same failure domain ###

A prometheus server should be deployed in the same failure domain as
the things it monitors.  This means the same data centre, region,
availability zone, etc.

This means that:

  * a prometheus shouldn't have to go over the public internet to
    scrape a service
      * (federation might go over public internet, though)
      
https://www.robustperception.io/scaling-and-federating-prometheus/

### There should normally be a single alertmanager for an organization ###

It seems that there should normally be a single alertmanager for an
organization, rather than having separate alertmanagers with different
configurations for each team or programme or division.  The best
reference I've found for this is from
[Julius Volz's Prometheus Design and Philosophy talk (7:44)](https://youtu.be/QgJbxCWRZ1s?t=7m43s)
which doesn't go into much detail as to *why* you'd do this.

## Security model

Prometheus has extensive
[documentation about its security model](https://prometheus.io/docs/operating/security/).
In particular, it distinguishes trusted users (who have admin access
to deploy and reconfigure prometheus) from untrusted users (who can
view prometheus metrics and dashboards).

Mostly, you control access to prometheus at the network level; there
is no support for authentication within prometheus.

You should be careful before enabling the `honor_labels` or
`--web.enable-admin-api` options, because both put extra trust in
scraped targets or untrusted users.

## Federation

Prometheus supports federation: a prometheus server can scrape the
metrics from another prometheus server.  This is to support some use
cases, such as:

  * have a global, coarse-grain view of the system
      * the global prometheus may have a coarser scrape resolution
      * the global prometheus
        [may aggregate instance-level time series into service-level](https://prometheus.io/docs/prometheus/latest/federation/#hierarchical-federation)

You may also limit the metrics collected at the global level to those
which you really care about (e.g. those which form a Service Level
Objective which impacts a Service Level Agreement you'd rather not
break)

In the event of an issue you could then debug further by querying the
local Prometheus server with a more granular view of the system.

## Alerting

A lot of Prometheus's philosophy on alerting is derived from
[Rob Ewaschuk's philosophy on alerting](https://docs.google.com/document/d/199PqyG3UsyXlwieHaqbGiWVa8eMWi8zzAn0YfcApr8Q/edit#)
which, after an editing process, became
[chapter 6 of the SRE book](https://landing.google.com/sre/book/chapters/monitoring-distributed-systems.html).

### Alert on symptoms, not causes

As mentioned above, Prometheus only deals with aggregated numerical
data, not raw event data.  Prometheus data is used to trigger alerts,
based on symptoms ("the system is broken") rather than causes.  From
Rob Ewaschuk's philosophy, he has a section
[cause-based alerts are bad (but sometimes necessary)](https://docs.google.com/document/d/199PqyG3UsyXlwieHaqbGiWVa8eMWi8zzAn0YfcApr8Q/edit#heading=h.6ammb5h32uqq).

### Access to context

However, people responding to the alert may well need access to raw
event data such as logs or exceptions to understand the underlying
causes in order to resolve the alert.

As [Brian Brazil put it](https://groups.google.com/forum/m/#!topic/prometheus-users/JZigzNa48QM):

> you get your alert, and [...] then jump to logs to see what exactly went wrong. This sort of debug-level information would never pass through Prometheus or the Alertmanager.

