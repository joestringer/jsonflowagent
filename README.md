JSONStats/JSONFlowAgent
=======================

JSONFlowAgent is a Net-SNMP subagent which retrieves OpenFlow switch statistics
from the JSONStats NOX component and pushes them to an AgentX-based SNMP master
agent.

JSONStats is a NOX component that exposes OpenFlow switch statistics via JSON.
Any interested clients can connect and retrieve the information by using a
simple request format. For more information, see section "Message Format".

Contents
--------

- Installation
- Running
- Message Format
- License
- Known Bugs/Workarounds
- Todo

Installation
------------

JSONStats builds on NOX "zaku" (0.9.0) and later. If you have recieved this copy
with RouteFlow, then JSONStats component will be compiled along with the rest of
the RouteFlow components (as part of NOX).

JSONFlowAgent requires the Net-SNMP headers to build. On Debian-based systems,
you will need to install the "libsnmp-dev" package. In the jsonflowagent
directory, type 'make' to compile. JSONFlowAgent has been tested against
Net-SNMP 5.4 and 5.7 series under Ubuntu 11.04.

On earlier versions of Net-SNMP, there may be compiler warnings about inline
functions from `/usr/include/net-snmp/agent/*`; Later versions compile cleanly.

Running
-------

JSONStats is a NOX Component, so to run it, just add 'jsonstats' to the end of
your nox command, eg:-

```
$ cd RouteFlow/rf-controller/build/src && ./nox_core --verbose=ANY:syslog:DBG \
-i ptcp:6633 routeflowc jsonstats
```

JSONFlowAgent is a Net-SNMP subagent, and as such it expects that the Net-SNMP
Master Agent is already running. You will need to enable agentX support and
agentX socket in /etc/snmp/snmpd.conf:

```
 master          agentx
 agentXSocket    tcp:localhost:705
```

In addition, you may need to modify the "Access Control" section of your
snmpd.conf to allow access to the CPqD.openflow MIB.

After compiling JSONFlowAgent, there will be an executable called "jfa". When
it runs, it will attempt to connect to the SNMP Master Agent, and to JSONStats.

To test that JSONFlowAgent is functioning correctly, try running snmpwalk:-

```
$ snmpwalk -Le -v2c -c public 127.0.0.1 .1.3.6.1.4.1.13727.2380
```

If you have trouble running JSONFlowAgent, open an issue on GitHub.

Message Format
--------------

Clients interested in retrieving openflow statistics can set up a TCP connection
to the JSONStats server (default port 2703) and send a message with the
following text in the body:

```
{
  "type":"jsonstats",
  "command":"features_request"
}
```

Supported commands are "features_request" and "port_stats_request".

Replies come in the following format:

```
{
  "type":"features_reply",
  "datapaths": [
    {
      "datapath_id": "16564197374866",
      "ports": [
        {
          // Structure equivalent to ofp_phy_port
          // Alternatively, ofp_port_stats (for "port_stats_request")
          "port_no":1,
          "hw_addr":"00:DE:AD:BE:EF:00",
          "name":"eth0",
          "config":0,
          "state":1,
          "curr":3714,
          "advertised":1026,
          "supported":3631,
          "peer":1026
        },
        ... // More port structures (if available)
      ]
    },
    ... // More datapaths (if available)
  ]
}
```

This closely mirrors the format in OpenFlow 1.0, but represented in standard
JSON. Integer values <= 32bits are represented as JSON 'Number', while 64bit
integers are rendered as the decimal representation in a JSON 'String'.

The "port_stats_request" response is similar, but equivalent to ofp_port_stats.

License
-------

JSONStats and JSONFlowAgent follow the license for RouteFlow. For more
information, see https://github.com/CPqD/RouteFlow/blob/master/LICENSE

Known Bugs/Workarounds
----------------------

### JSONStats

- Other NOX apps requesting statistics will interfere with stats collection:
If a seperate NOX app sends a request for a particular OpenFlow stats message
while a JSONStats request is being handled, it is possible that JSONStats will
recieve two of the same OpenFlow stats reply from the same datapath. Currently
JSONStats does not keep track of which datapaths have replied to a given stats
request, so it will encode the duplicate response after the first. If JSONStats
is looking for replies from two datapaths, and one datapath sends two replies,
then it will aggregate those and respond to the client without stats from our
second datapath.

### JSONFlowAgent

- Statistics may be stale:
If we recieve statistics for a new datapath, then our mib handlers reserve
space for that datapath and its corresponding netsnmp_table_rows. Values
updated using these structures will stick until they are later updated. There
is currently no way to tell if the statistics we serve are stale.

- OID Path used by mib section handlers has an extra node:
In `mib/port_stats.cc`, `mib/phy_port.cc` we specify the CPqD->openflow OID path.
Something in Net-SNMP adds an extra child oid ".1" after this path. Specific
values for each of those OF statistic structures are then represented, followed
by datapath and port number.

- Parsing JSON may cause JFA to segfault:
When JFA receives invalid JSON responses, it is known to cause segfaults. At
some point it would be nice to find out why, and perhaps consider using a
different JSON-decode library rather that cJSON, but as we control both ends
of the communication for NOX jsonstats<->jsonflowagent, this library is
"good enough" for the moment.

- Net-SNMP timers are ignored:
When using the Net-SNMP API with snmp_select_info(), we are meant to pass the
select() parameters to snmp_select_info() before we hand them to select().
However, as at Net-SNMP 5.7.1 there is a bug that causes snmp_select_info() to
change the timeout to 1usec. We work around this by passing snmp_select_info()
a dummy timeval struct and ignoring it.

Todo
----

- Fix known bugs
- Commandline argument parsing for
|- polling frequency
|- jsonstats "server:port"
|- agentX "server:port"
- More OFP stats support -- Aggregate, Flows, Tables, etc.
