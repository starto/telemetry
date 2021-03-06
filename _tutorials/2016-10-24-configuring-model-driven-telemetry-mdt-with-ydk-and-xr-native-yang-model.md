---
published: true
date: '2016-10-24 16:39 +1100'
title: Configuring Model-Driven Telemetry (MDT) with YDK and XR Native YANG model
author: Marco Umer
excerpt: Configuring Model-Driven Telemetry with YDK and XR Native YANG model
tags:
  - iosxr
  - YDK
  - YANG
  - Telemetry
---

{% include toc icon="table" title="Configuring MDT with YDK + XR native YANG" %}
{% include base_path %}

## Getting a complete MDT configuration with Native YANG and YDK

In an [earlier tutorial](https://xrdocs.github.io/telemetry/tutorials/2016-08-08-configuring-model-driven-telemetry-with-ydk/), Shelly introduces a methodology to configure MDT using YDK and the OpenConfig Telemetry YANG model.

This document provides a similar tutorial, addressing a complete XR Telemetry TCP dial-out configuration using the native YANG model and YDK.

I have tested the configuration in this document using the relative recent Vagrant IOS-XRv box version 6.2.1.15I.
If you are looking for more information on IOS-XRv for Vagrant, you may follow Akshat tutorial [IOS-XRv Vagrant Quick Start](https://xrdocs.github.io/application-hosting/tutorials/iosxr-vagrant-quickstart) for step by step instructions.

If you are not familiar with IOS-XRv, follow Akshat tutorial [IOS-XRv Vagrant Quick Start](https://xrdocs.github.io/application-hosting/tutorials/iosxr-vagrant-quickstart) for step by step instructions.

## Tutorial goal

By the end of this tutorial, you will have implemented the following configuration using YDK on the router under testing. If you are unfamiliar with the configuration just check the following tutorial [Configuring Model-Driven Telemetry (MDT)](https://xrdocs.github.io/telemetry/tutorials/2016-07-21-configuring-model-driven-telemetry-mdt/)

{% capture "output" %}
CLI Output:

```
test_XR#show running-config telemetry model-driven
Fri Oct 21 06:51:06.926 UTC
telemetry model-driven
 destination-group DG_Test
  address family ipv4 192.168.10.3 port 5432
   encoding self-describing-gpb
   protocol tcp
  !
 !
 sensor-group SG_Test
  sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
 !
 subscription 1
  sensor-group-id SG_Test sample-interval 30000
  destination-id DG_Test
 !
!
```

{% endcapture %}

<div class="notice--warning">
{{ output | markdownify }}
</div>

## Connect to the router and import YDK's libraries

As described in previous YDK tutorials, we are importing the YDK Netconf library to communicate with the router.  
We also import the CRUDService YDK's library (taking care of create, read, update and delete YDK objects from the router) and the IOS_XR native YDK model.  
The Empty type that we import from ydk.types has a special purpose later in the document to signal with its presence, the request to activate the submitted subscription.

```python
from ydk.providers import NetconfServiceProvider
from ydk.services import CRUDService
from ydk.types import Empty
import ydk.models.cisco_ios_xr.Cisco_IOS_XR_telemetry_model_driven_cfg as xr_telemetry

HOST = '192.168.10.2'
PORT = 830
USER = 'vagrant'
PASS = 'vagrant'

xr = NetconfServiceProvider(address=HOST,
	port=PORT,
	username=USER,
	password=PASS,
	protocol = 'ssh')
```

With that, we are now connected to the router:

{% capture "output" %}
CLI Output:

```
RP/0/RP0/CPU0:test_XR#show netconf-yang clients
Fri Oct 21 01:36:02.850 UTC
Netconf clients
client session ID|     NC version|    client connect time|        last OP time|        last OP type|    <lock>|
       4261169968|            1.1|         0d  0h  0m 28s|                    |                    |        No|
RP/0/RP0/CPU0:test_XR#

```

{% endcapture %}

<div class="notice--info">
{{ output | markdownify }}
</div>


## Define and apply the destination group

To access the native XR telemetry YANG model used in this tutorial (Cisco-IOS-XR-telemetry-model-driven-cfg.yang) use the [YANG public repository](https://github.com/YangModels/yang/tree/master/vendor/cisco/xr).  
Explore this same repository to see the other vendors and standard model YANGs but just in case you are looking for the OpenConfig, you will have to use [YANG OpenConfig repository](https://github.com/openconfig/public).

To explore the telemetry YANG model read directly the yang file or for example, follow a friendlier  tree that the Pyang utility generates.  
Note: if you have installed the YDK environment, you can use Pyang from this environment as `pyang -f tree Cisco-IOS-XR-telemetry-model-driven-cfg.yang`

{% capture "output" %}
PYANG output for the destination-groups:

```
module: Cisco-IOS-XR-telemetry-model-driven-cfg
   +--rw telemetry-model-driven
      <SNIP>
      +--rw destination-groups
      |  +--rw destination-group* [destination-id]
      |     +--rw destinations
      |     |  +--rw destination* [address-family]
      |     |     +--rw address-family    Af
      |     |     +--rw ipv4* [ipv4-address destination-port]
      |     |     |  +--rw ipv4-address        inet:ip-address-no-zone
      |     |     |  +--rw destination-port    xr:Cisco-ios-xr-port-number
      |     |     |  +--rw protocol!
      |     |     |  |  +--rw protocol        Proto-type
      |     |     |  |  +--rw tls-hostname?   string
      |     |     |  |  +--rw no-tls?         int32
      |     |     |  +--rw encoding?           Encode-type
      |     |     +--rw ipv6* [ipv6-address destination-port]
      |     |        +--rw ipv6-address        xr:Cisco-ios-xr-string
      |     |        +--rw destination-port    xr:Cisco-ios-xr-port-number
      |     |        +--rw protocol!
      |     |        |  +--rw protocol        Proto-type
      |     |        |  +--rw tls-hostname?   string
      |     |        |  +--rw no-tls?         int32
      |     |        +--rw encoding?           Encode-type
      |     +--rw destination-id    xr:Cisco-ios-xr-string


```  
{% endcapture %}

<div class="notice--warning">
{{ output | markdownify }}
</div>

This is what the YANG model maps to YDK code for our example:

```python
dgroup=xr_telemetry.TelemetryModelDriven.DestinationGroups.DestinationGroup()
dgroup.destination_id="DG_Test"
dgroup.destinations=dgroup.Destinations()

new_destination=dgroup.Destinations.Destination()
new_destination.address_family=xr_telemetry.AfEnum.IPV4

new_ipv4=xr_telemetry.TelemetryModelDriven.DestinationGroups.DestinationGroup().Destinations().Destination().Ipv4()
new_ipv4.destination_port=5432
new_ipv4.ipv4_address="192.168.10.3"
new_ipv4.encoding=xr_telemetry.EncodeTypeEnum.SELF_DESCRIBING_GPB
new_ipv4.protocol=xr_telemetry.TelemetryModelDriven.DestinationGroups.DestinationGroup().Destinations().Destination().Ipv4().Protocol()
new_ipv4.protocol.protocol=xr_telemetry.ProtoTypeEnum.TCP
new_destination.ipv4.append(new_ipv4)
dgroup.destinations.destination.append(new_destination)

```

Once you’ve populated the object, we can apply it to the router using the create method on the CRUDService object from YDK:

```python
rpc_service = CRUDService()
rpc_service.create(xr, dgroup)
```

And here is the expected CLI output with the destination group describing where and how to steam:

{% capture "output" %}
CLI Output:

```
RP/0/RP0/CPU0:test_XR#sh running-config telemetry model-driven
Fri Oct 21 06:35:06.731 UTC
telemetry model-driven
 destination-group DG_Test
  address family ipv4 192.168.10.3 port 5432
   encoding self-describing-gpb
   protocol tcp
  !
 !
!
```

{% endcapture %}

<div class="notice--info">
{{ output | markdownify }}
</div>


## Define and apply the sensor group

As for the destination group, this is the section of the YANG model for the sensor group:

{% capture "output" %}
PYANG output for the sensor-groups:

```
module: Cisco-IOS-XR-telemetry-model-driven-cfg
  +--rw telemetry-model-driven
	  +--rw sensor-groups
      |  +--rw sensor-group* [sensor-group-identifier]
      |     +--rw sensor-paths
      |     |  +--rw sensor-path* [telemetry-sensor-path]
      |     |     +--rw telemetry-sensor-path    string
      |     +--rw enable?                    empty
      |     +--rw sensor-group-identifier    xr:Cisco-ios-xr-string
	<SNIP>

```  
{% endcapture %}

<div class="notice--warning">
{{ output | markdownify }}
</div>

... and the YDK code that it maps to:

```python
sgroup = xr_telemetry.TelemetryModelDriven.SensorGroups.SensorGroup()
sgroup.sensor_group_identifier="SG_Test"

sgroup.sensor_paths = sgroup.SensorPaths()
new_sensorpath = sgroup.SensorPaths.SensorPath()
new_sensorpath.telemetry_sensor_path = 'Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters'
sgroup.sensor_paths.sensor_path.append(new_sensorpath)
```

Now we are ready to submit the sensor group object just populated with the same `create` method from the CRUDService object:

```python

rpc_service.create(xr, sgroup)

```

Note: you need to initialize the rpc_service as `rpc_service = CRUDService()` a single time.  
If you remember, we have already done it when creating the destination group but if you skipped the previous step, add it before requesting to create for the sensor group.

Let's check the CLI running-configuration again. You should now find the destination (from the previous step) and sensor group just created in place:

{% capture "output" %}
CLI Output:

```
RP/0/RP0/CPU0:test_XR#show running-config telemetry model-driven
Fri Oct 21 06:45:58.444 UTC
telemetry model-driven
 destination-group DG_Test
  address family ipv4 192.168.10.3 port 5432
   encoding self-describing-gpb
   protocol tcp
  !
 !
 sensor-group SG_Test
  sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
 !
!
```

{% endcapture %}

<div class="notice--info">
{{ output | markdownify }}
</div>


## Define and apply the subcription

The final step in our configration is the subcription.

{% capture "output" %}
PYANG output for the subcription:

```
module: Cisco-IOS-XR-telemetry-model-driven-cfg
  +--rw telemetry-model-driven
	<SNIP>
      +--rw subscriptions
      |  +--rw subscription* [subscription-identifier]
      |     +--rw source-address!
      |     |  +--rw address-family    Af
      |     |  +--rw ip-address?       inet:ipv4-address-no-zone
      |     |  +--rw ipv6-address?     string
      |     +--rw sensor-profiles
      |     |  +--rw sensor-profile* [sensorgroupid]
      |     |     +--rw sample-interval?      uint32
      |     |     +--rw heartbeat-interval?   uint32
      |     |     +--rw supress-redundant?    empty
      |     |     +--rw sensorgroupid         xr:Cisco-ios-xr-string
      |     +--rw destination-profiles
      |     |  +--rw destination-profile* [destination-id]
      |     |     +--rw enable?           empty
      |     |     +--rw destination-id    xr:Cisco-ios-xr-string
      |     +--rw source-qos-marking?        uint32
      |     +--rw subscription-identifier    xr:Cisco-ios-xr-string

	<SNIP>

```  
{% endcapture %}

<div class="notice--warning">
{{ output | markdownify }}
</div>

This is how that ends up in YDK code:

```python
sub = xr_telemetry.TelemetryModelDriven.Subscriptions.Subscription()
sub.subscription_identifier = "1"

sub.sensor_profiles = sub.SensorProfiles()
sub.destination_profiles = sub.DestinationProfiles()

new_sprofile = sub.SensorProfiles.SensorProfile()
new_sprofile.sensorgroupid = 'SG_Test'
new_sprofile.sample_interval = 30000

new_dprofile = sub.DestinationProfiles.DestinationProfile()
new_dprofile.destination_id="DG_Test"
new_dprofile.enable=Empty()

sub.sensor_profiles.sensor_profile.append(new_sprofile)
sub.destination_profiles.destination_profile.append(new_dprofile)
```

```python
rpc_service.create(xr, sub)
```

If you check your router running-configuration, you should now have a complete TCP dial-out telemetry configuration  as:

{% capture "output" %}
CLI Output:

```
P/0/RP0/CPU0:test_XR#show running-config telemetry model-driven
Fri Oct 21 06:51:06.926 UTC
telemetry model-driven
 destination-group DG_Test
  address family ipv4 192.168.10.3 port 5432
   encoding self-describing-gpb
   protocol tcp
  !
 !
 sensor-group SG_Test
  sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
 !
 subscription 1
  sensor-group-id SG_Test sample-interval 30000
  destination-id DG_Test
 !
!
```

{% endcapture %}

<div class="notice--info">
{{ output | markdownify }}
</div>

## Clean

One last command to delete the configuration:

```python
rpc_service.delete(xr, xr_telemetry.TelemetryModelDriven())
```
And confirm in the router's 'show running-config'

{% capture "output" %}
CLI Output:

```
RP/0/RP0/CPU0:test_XR#show running-config telemetry model-driven
Fri Oct 21 06:59:46.743 UTC
% No such configuration item(s)

RP/0/RP0/CPU0:test_XR#

```

{% endcapture %}

<div class="notice--info">
{{ output | markdownify }}
</div>

## Conclusion

This tutorial complements the concepts explained by Shelly in her [earlier tutorial](https://xrdocs.github.io/telemetry/tutorials/2016-08-08-configuring-model-driven-telemetry-with-ydk/).
The same values in programmability using YANG models, automatic generation of Python classes that inherit the syntactic checks and requirements of the underlying model, while also handling all the details for the sensor group using the native telemetry YANG model.
