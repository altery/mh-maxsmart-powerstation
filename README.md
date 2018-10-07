# mh-maxsmart-powerstation
Protocol to control the Max Hauri MaxSMART Powerstation (SOP111)

The [MaxSMART Powerstation](https://www.maxsmart.ch/power-station.html) is a plug assembly that may be controlled using a mobile app. It features controllable sockets including
schedules, power metering, presence simulation and master-slave configuration.

Unfortunately, there is no documentation available of the protocol that controls
the Powerstation. And the manufacturer [has no intention](https://www.maxsmart.ch/faq1.html)
to change that.

As the hardware is a bit outdated, it is currently available for a very attractive
price. So I took the shot, and tried to integrate the Powerstation in my
Loxone/Node-RED setup, reverse-engineering the communication protocol of the
Powerstation.

This repository documents the findings.

# Phoning home, time and firmware updates

*Important: The MaxSMART 1.0 product line seems to be end of life. The new MaxSMART 2.0
devices use a different protocol. It might be possible that the Powerstation
may supplied with a firmware upgrade (using the new maxSMART 2.0 app?) that makes
it compatible with MaxSMART 2.0 - but incompatible with the protocol described here.*

Immediately after the Powerstation receives an IP address, the devices contacts
a server of the manufacturer.

It performs a DNS-lookup of www.maxsmart.ch. It then sends a datagram to the
resolved IP:

| Property         | Value         |
| ---------------- | --------------|
| Protocol         | UDP |
| Source IP        | Powerstation IP  |
| Destination IP   | 217.192.96.184 |
| Source Port      | 8888           |
| Destination Port | 5000 |
| Payload          | See payload   |

Payload:
```
50SWP1234000000000{"cpuid":"AABBCCDDEEFF554433221100","sak":"0011AABBCCDD"}
```

The server responds with two UDP datagrams to the source port 8888.

The first datagram (prefix `50`) contains the current date and time and the 'latest' firmware version available:
```
50SWP1234000000000{"register":0,"url":"www.maxsmart.ch","port":5000,"time":"2018-10-12,23:15:12","nver":"1.30"}
```

The time field is used to configure the real time clock of the Powerstation. This is important for all time based operations (like rules).

The second datagram (prefix `51`) contains a link to a firmware image:
```
51SWP1234000000000{"ver":"1.24","url":"http://www.maxsmart.ch/cloud/downloads/b87bd84a-a11f-42f6-8d2c-933a15d09a4e.bin","checksum":6676180}
```
The device can probably update itself using such an image.

**This mechanism seems to be highly insecure. An attacker could easily spoof DNS requests and present a fake firmware to the device. Also, all transmissions are in plaintext, and there doesn't seem to be any signature on the firmware image.**

It is not yet determined, if the device automatically installs updates or if this
has to be initiated by the user. Also, the algorithm to calculate the checksum is unknown.

If there is no cloud account setup in the MH app, the device does not communicate
with the internet after the initial exchange. If the user enters MH cloud
account credentials within the MH app, the device is immediately added to the cloud account and the device is reported as registered:

```
50SWP1234000000000{"register":1,"url":"www.maxsmart.ch","port":5000,"time":"2018-10-13,00:31:04","nver":"1.30"}
```
**After that, it heavily exchanges unencrypted and unsigned UDP messages with the server of the manufacturer which allows complete control of the device from within the internet. Everything that can locally be done using the HTTP interface, can also be done remotely without *ANY* authentication using simple UDP datagram.**

It is a matter of simply spoofing DNS responses in order to get complete control of devices from the MaxSmart 1.0 range. I suspect it is also possible to brick the device by providing corrupt firmware updates.

# Protocol
## Discovery Protocol
The MaxSMART devices support a proprietary discovery protocol based on UDP.

### Request

The discovery request is a simple UDP datagram. Typically, the datagram is sent to the
broadcast address of the network:

| Property         | Value         | Remarks                                             |
| ---------------- | --------------| ----------------------------------------------------|
| Protocol         | UDP           |                                                     |
| Source IP        | Client IP     | Discovery responses will be sent to this IP         |
| Destination IP   | 192.168.0.255 | Example for network 192.168.0.0/24                  |
| Source Port      | any           | Discovery responses will be sent to this IP         |
| Destination Port | 8888          |                                                     |
| Payload          | See payload   |The date part seems to be optional.                  |

#### Request payload

The payload is a string consisting of two fields:
```
00dv=all,2018-10-07,10:48:42,13;
```
The 00dv might be some sort of device selector. The purpose of the date/time field is unclear. It seems to be optional. At least the Powerstation responds, even if the field is missing completely:
```
00dv=all;
```

### Response

The devices respond to the request by sending a datagram to the source port of the
request datagram:

| Property         | Value         | Remarks                                             |
| ---------------- |---------------| ----------------------------------------------------|
| Protocol         | UDP           |                                                     |
| Source IP        | PowerStation IP | Discovery responses will be sent to this IP       |
| Destination IP   | Client IP     | Example for network 192.168.0.0/24                  |
| Source Port      | 8888          | The source port will be used to send responses to.  |
| Destination Port | Request src port |                                                  |
| Payload          | See payload   | <p>                                                 |

#### Response payload
The response payload is a JSON formatted document:
```
{
    "response":0,
    "data":{
       "sn":"SWP1234000000000",
       "name":"STRIP",
       "pname":["PORT1","PORT2","PORT3","PORT4","PORT5","PORT6"],
       "mac":"BC:2B:00:00:00:00",
       "sak":"123456789ABC",
       "ver":"1.30",
       "register":0,
       "nver":"1.30",
       "protect":0
     }
}
```

##### Data fields
| Field            | Description                                            |
| ---------------- |--------------------------------------------------------|
| sn               | Serial (as printed on the backside)                    |
| name             | Type of device. Powerstation is `STRIP`                |
| pname            | Enumerates the name of the ports (sockets) of the device. Note: The names are customizable. Discovery is the only way to query the custom port names!|
| mac              | The ethernet MAC address                               |
| sak              | Unknown. Might be some sort of credential. Required to add the device to the MaxSmart cloud.        |
| ver              | Firmware version.                                      |
| register         | Whether the device is added to a cloud account on https://www.maxsmart.ch/cloud/                                                |
| nver             | Probably the *new* firmware version, if there is an update available.                |
| protect          | unknown                                                |

*The IP address of the device is not explicitly sent in the payload. But the client
may use the source IP address of the datagram.*

## HTTP Protocol
The Powerstation uses a protocol that is piggybacked on `HTTP GET` request. Note
that the API is *not* RESTful though.

Note: There is a second UDP based protocol that has the same functionality as the HTTP version.

### Common request structure

#### Requests

Every request to the device is sent as `HTTP GET` request to the IP address determined from discovery and the HTTP standard port `80`:
```
GET http://<IP>/?cmd=<COMMAND>&json=<PARAMS>
```
The `cmd` parameter is a number. The commands are split into actual *actions*, that modify the state of the device
and *queries*, that are for informational purpose. Actions use commands in the `2xx`
range, whereas queries use commands in the `5xx` range.

The `json` query parameter is a [URL encoded JSON document](https://www.w3schools.com/tags/ref_urlencode.asp) that contains the arguments of the command.

#### Example
```
GET http://<IP>/?cmd=200&json=%7B%22sn%22:%22SWP1234000000000%22,%22port%22:1,%22state%22:0%7D
```

#### Response

The HTTP status code of every response is `200` (i.e. not used for application logic).

The payload of the HTTP response is a JSON document. Note that the device incorrectly
responds with a content type of `text/plain` instead of `application/json`.

The general structure is as follows:

```
{
  "response": 510,
  "code" : 200,
  "data" : <command specific>
}
```

| Field            | Description                                            |
| ---------------- |--------------------------------------------------------|
| response         | The command code of the request ("response to")        |
| code             | The result of the command.<br>`200`: Success<br>`400`: Fail<br>Note: In case of request argument errors, the device may just omit sending a response altogether.|
| data             | Command specific data                              |

Because the `response` and `code` envelope are identical for all commands, only
the `data` field is documented in the following section. Also, only the "happy path"
(command executed successfully) is documented.

### Commands

#### Querying commands
* [`502` Query device date and time](cmd-502.md)
* [`510` Query energy consumption statistics](cmd-510.md)
* [`511` Query port status](cmd-511.md)
* [`512` Query master/slave port configuration](cmd-512.md)
* [`513` Query port status](cmd-513.md)
* [`515` Query timer configuration](cmd-515.md)
* [`516` Query rules](cmd-516.md)
* [`517` Query random generator](cmd-517.md)

#### Modifying commands
* [`120` Perform factory reset](cmd-120.md)
* [`200` Switch ports](cmd-200.md)
* [`201` Set port names](cmd-201.md)
* [`202` Configure master/slave ports](cmd-202.md)
* [`204` Set timer](cmd-204.md)
* [`205` Configure energy costs](cmd-205.md)
* [`206` Configure rules](cmd-206.md)
* [`207` Configure random generator](cmd-207.md)
