# Siber SKY eBUS

This project contains files and instructions to manage Siber Sky MFV
using eBUS protocol. First of all, this proposal is composed of four
principal components.

1. A hardware device that reads and writes on the eBUS.
2. A raspberry pi executing `ebusd` daemon configured to read and
   write Siber commands.
3. A MQTT broker that will receive `ebusd` messages.
4. A domotic control software that will consume `ebusd` MQTT messages
   and will add automation and monitoring.
   
This recipe will discuss about (1) and (2) assuming knowledge about
eBUS and how it can be configured. So, in order to start, it is
important to read some documentation first:

- [eBUS documentation](https://ebusd.eu/).
- [MQTT documentation](https://mqtt.org/).
- [Mosquitto MQTT broker](https://mosquitto.org/).

**WARNING!!!** All the information contained in this project comes
without any warranty, it is not covered by Siber official
documentation, and not officially supported by anyone.

# Hardware

First of all, it is important to notice that [Siber Sky
2](https://www.siberzone.es/descarga/siber-df-sky-2-10416/) machines
are connected to their control panel via eBUS protocol using just a
2-wire connection. You will find in the link information about how the
machine is connected to the control ([MANUAL CONEXIÓN
MANDO](https://www.siberzone.es/Media/Uploads/dlm_uploads/siber-ventilacion-manual-conexion-mando-dfexctrln-1.pdf)).

It is possible to connect in parallel to the Siber control another
eBUS device that will allow us to read/write commands into the bus. I
have used this one:

- [eBUS coupler USB](https://esera.de/Produkte/eBus/135/1-Wire-Hub-Platine).

Which could be quite slow, but works well for me. It can be connected
in parallel with the control panel of Siber. The connection with the
USB coupler requires a device, in my case a raspberry pi, that will
connect via USB to the coupler.

# Software

To make everything work, you need to install
[ebusd](https://ebusd.eu/) in the raspberry pi. In my case, the
`ebusd` daemon is executed as:

```
/usr/bin/ebusd --enablehex -c /home/siber/configs -p 8888 -l /var/log/ebusd.log --mqtthost 192.168.1.77 --mqttport 1883 --mqttqos 2
```

In order to achieve that execution line, it is necessary to copy in
`/etc/default/ebusd` the following content:


```
# /etc/default/ebusd:
# config file for ebusd service.

# Options to pass to ebusd (run "ebusd -?" for more info):
EBUSD_OPTS="--enablehex -c /home/siber/configs -p 8888 -l /var/log/ebusd.log --mqtthost 192.168.1.77 --mqttport 1883 --mqttqos 2"

# MULTIPLE EBUSD INSTANCES WITH SYSV
# In order to run multiple ebusd instances on a SysV enabled system, simply
# define several EBUSD_OPTS with a unique suffix for each. Recommended is to
# use a number as suffix for all EBUSD_OPTS settings. That number will then be
# taken as additional "instance" parameter to the init.d script in order to
# start/stop an individual ebusd instance instead of all instances.
# Example: (uncomment the EBUSD_OPTS above)
#EBUSD_OPTS1="--scanconfig -d /dev/serial/by-id/usb-FTDI_FT232R_USB_UART_A50285BI-if00-port0 -p 8888 -l /var/log/ebusd1.log"
#EBUSD_OPTS2="--scanconfig -d /dev/serial/by-id/usb-FTDI_FT232R_USB_UART_A900acTF-if00-port0 -p 8889 -l /var/log/ebusd2.log"
#EBUSD_OPTS3="--scanconfig -d /dev/serial/by-id/usb-FTDI_FT232R_USB_UART_A900beCG-if00-port0 -p 8890 -l /var/log/ebusd3.log"

# MULTIPLE EBUSD INSTANCES WITH SYSTEMD
# In order to run muiltiple ebusd instances on a systemd enabled system, just
# copy the /lib/systemd/system/ebusd.service file to /etc/systemd/system/
# with a different name (e.g. ebusd-2.service), remove the line starting with
# 'EnvironmentFile=', and replace the '$EBUSD_OPTS' with the options for that
# particular ebusd instance.
```

And in `/home/siber/configs` there is the `3c.siber.csv` file as it is
in this project `configs` directory. Once the `ebusd` and the MQTT
server are ready, it can be restarted so it can take into account the
content of the `3c.siber.csv` file. Once everything is running, you
should be able to subscribe to `ebusd` MQTT messages:

```
$ mosquitto_sub -h localhost -v -t 'ebusd/#'
ebusd/global/version ebusd 22.3.p20220508
ebusd/global/running true
ebusd/global/signal true
ebusd/global/updatecheck version 22.4 available
ebusd/Siber/indoor_temperature/get (null)
ebusd/Siber/outdoor_temperature/get (null)
ebusd/Siber/supply_flow/get (null)
ebusd/Siber/exhaust_flow/get (null)
ebusd/Siber/supply_pressure/get (null)
ebusd/Siber/exhaust_pressure/get (null)
ebusd/Siber/flow_imbalance/get (null)
ebusd/Siber/air_flow_level_1/get (null)
ebusd/Siber/bypass_status/get (null)
ebusd/Siber/indoor_temperature 21.1
ebusd/Siber/outdoor_temperature 10.3
ebusd/Siber/supply_flow 90
ebusd/Siber/exhaust_flow 90
ebusd/Siber/supply_pressure 110.0
ebusd/Siber/exhaust_pressure 70.0
ebusd/Siber/flow_imbalance 0;-50;50;1;0
ebusd/Siber/air_flow_level_1 90;50;200;5;100
ebusd/Siber/bypass_status CLOSED
ebusd/global/uptime 11667419
ebusd/Siber/indoor_temperature 21.1
ebusd/Siber/outdoor_temperature 10.3
ebusd/Siber/supply_flow 90
ebusd/Siber/exhaust_flow 90
ebusd/Siber/bypass_status CLOSED
ebusd/Siber/supply_pressure 110.0
ebusd/Siber/exhaust_pressure 70.0
ebusd/Siber/air_flow_level_1 90;50;200;5;100
ebusd/Siber/flow_imbalance 0;-50;50;1;0
<...>
```

Adding the MQTT messages to a domotic control as OpenHAB (I'm using
it) or Home Assistant should be easy. Even more, `ebusd` has an
integration with Home Assistant, which I do not know. In OpenHAB there
is another integration of `ebusd`, but I prefered to use the MQTT
messages instead of the integration.

There is just one major drawback. The Siber control panel writes
periodically the `fan_operation` level (MOISTURE, NORMAL, HIGH,
INTENSE), and so, any attempt to change the `fan_operation` will
eventually be override by the Siber control. However, it can be
overcome easily just by using only one `fan_operation` (NORMAL) in the
Siber control and changing directly the `airflow_1`.

# Automations

## Fan Operation

As it was mentioned above, I needed to add fan operation levels that,
instead of operating the fan just operate the `airflow_1` which
corresponds to the fan level I have running in the Siber Control
panel. So, my fan levels are different flows I have configured inside
OpenHAB.

## Bath Humidity

When the humidity in the bathroom increases a lot, it is likely
because someone is having a bath. In that case, the air flow is
increased to avoid moisture in the bathroom. Once the humidity is
reduced and returns to normal values, the air flow returns to its
previous position.

## Summer Ventilation

When in summer over night the bypass is open, and there is a chance
for supplying cold air, the air flow increases over night to take
advantage of the situation and cool the house by a small price.

## Cooking

When the kitchen is being used, and the air extractor running, an
imbalance between supply and exhaust is introduced, so there is more
supplied air than exhausted. This way, the extractor seem to work
better.
