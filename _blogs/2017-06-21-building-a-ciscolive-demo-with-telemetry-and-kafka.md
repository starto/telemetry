---
published: false
date: '2017-06-21 10:22 -0600'
title: Building a CiscoLive Demo with Telemetry and Kafka
---
## Behind the Scenes of Continuous Automation

Every year at Cisco Live, my team helps put together the demos that go into the World of Solutions. Geeks that we are, we get a thrill out of showing off the art of the possible. But the real purpose of a demo is to start a conversation with the network engineers and operators who stop by our booth.     

This year, my colleague asked me to help integrate telemetry into a demo called "Continuous Automation." The goal of the demo is to illustrate how model-driven telemetry (MDT) can be used with model-driven APIs to automate a simple provisioning and validation task (it's based a real customer use case that we're currently working on).  

Pieces of the demo were already in place: a small Python app that utilized the YDK Python APIs to configure a BGP neighbor and execute a connectivity test (ping) from the router when the neighbor came up.  The problem was that the app had no way to know when the neighbor came up.  Enter MDT!

## The Easy Part: Data Model and Router Config
The operational data that we needed was the BGP neighbor session state.  This is easily available in the OpenConfig BGP model:

```module: openconfig-bgp
   +--rw bgp
      +--rw neighbors
         +--rw neighbor* [neighbor-address]
            +--ro state
               +--ro session-state?   enumeration
```

Translating this into an MDT sensor path config looks like this:

```
telemetry model-driven
 sensor-group BGP
  sensor-path openconfig-bgp:bgp/neighbors/neighbor/state
```

Adding a destination group and a subscription gets the router streaming out the needed data in a Google Protocol Buffer (GPB) over TCP:

```
telemetry model-driven
 destination-group G1
  address-family ipv4 198.18.1.127 port 5432
   encoding self-describing-gpb
   protocol tcp
  !
 subscription S1
  sensor-group-id BGP sample-interval 5000
  destination-id G1
```

 But then what?  How do you get data from a TCP stream into a Python app?
 
 ## The Other Easy Part: Pipeline and Kafka
My go-to tool for consuming MDT data is pipeline, an open source utility that I've written about before.  If you've read my [previous tutorial](https://xrdocs.github.io/telemetry/tutorials/2016-10-03-pipeline-to-text-tutorial/), you'll recognize the  ```[testbed]``` input stage in the default pipeline.conf.  This will work "as-is" with the destination-group above.

{% capture "output" %}

```
[testbed]
stage = xport_input
type = tcp
encap = st
listen = :5432
```  
{% endcapture %}

<div class="notice--warning">
{{ output | markdownify }}
</div>

That's good for input, but what about output?  Pipeline can write data to three destinations:
- a file
- time series databases like InfluxDB
- Kafka

Writing to a file would probably work (Python has extensive file handling capabilities) but it seemed clumsy.  Writing to InfluxDB would also have worked (I could use Python REST packages to query the database) but seemed too heavy weight for a simple demo.  That left me with Kafka.  I've been wanting to do a Kafka demo for a while and there are Python packages to work with Kafka, so I figured...why not?  If nothing else, I'll learn something new.

For pipeline to output to Kafka, all you have to do is uncomment the following lines in the ```[mykafka]``` section of the default pipeline.conf. In the example below, I'm running Pipeline and Kafka on the same machine, so I used the broker address of "localhost" and the topic called "telemetry."

{% capture "output" %}

```
[mykafka]
stage = xport_output
type = kafka
encoding = json
brokers = localhost:9092
topic = telemetry
```  
{% endcapture %}

<div class="notice--warning">
{{ output | markdownify }}
</div>

# The Easiest Part
Since I haven't installed Kafka before, I thought it might be hard.  But it couldn't have been easier.  I followed the first two steps in the [Apache Kafka Quickstart](https://kafka.apache.org/quickstart) guide.  Boom.  Done.  Didn't even have to alter the default properties files for Kafka and Zookeeper.