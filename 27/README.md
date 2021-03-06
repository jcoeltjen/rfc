---
domain: rfc.zeromq.org
shortname: 27/ZAP
name: ZeroMQ Authentication Protocol
status: stable
editor: Pieter Hintjens <ph@imatix.com>
---

This document specifies ZAP, the ZeroMQ Authentication Protocol. The use case for ZAP is a set of *servers* that need authentication of remote clients, and which talk to an *handler* that checks client credentials. ZAP defines how the servers connect to the handler, and the messages they exchange.

See also: http://rfc.zeromq.org/spec:23/ZMTP, http://rfc.zeromq.org/spec:24/ZMTP-NULL, http://rfc.zeromq.org/spec:24/ZMTP-PLAIN, http://rfc.zeromq.org/spec:25/ZMTP-CURVE, http://rfc.zeromq.org/spec:26/CURVEZMQ.

## Preamble

Copyright (c) 2013 iMatix Corporation.

This Specification is free software; you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation; either version 3 of the License, or (at your option) any later version. This Specification is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details. You should have received a copy of the GNU General Public License along with this program; if not, see <http://www.gnu.org/licenses>.

This Specification is a [free and open standard](http://www.digistan.org/open-standard:definition) and is governed by the Digital Standards Organization's [Consensus-Oriented Specification System](http://www.digistan.org/spec:1/COSS).

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](http://tools.ietf.org/html/rfc2119).

## Overall Design

### What Problems does ZAP Address?

When a server needs to authenticate a client it has to choose between either doing the authentication itself, or calling on an external party to do this work. While built-in authentication is simple to understand and build, it is limited in many ways:

* It cannot easily be extended to arbitrary security mechanisms.
* It cannot integrate with external authentication systems.
* It tends towards customized designs so that each product has its own approach.

For security administrators, the ideal design has all applications calling on the same central repository of credentials. The problem then becomes, "how do we standardize this interaction?"

As in all distributed designs, ZeroMQ already provides a large part of the answer by defining a common message-based transport layer that can be used in any language and on any platform. The second part of the answer is a simple protocol on top of ZeroMQ.

ZAP is such a protocol, specifically aimed at authentication.

### Reducing The Component Count

One undesirable side-effect of building distributed applications is that the number of components increases. This makes it harder to build products that are simple to install and use. If we designed ZAP naively to use a TCP transport and external handler, we would end up with additional services to install and run even for the simplest application.

So, ZAP uses an *inprocess bridge* design. That is, ZAP itself requires that the handler run as a thread within the same process as the servers. When an application uses multiple processes, each process will contain a ZAP handler.

A ZAP handler can then talk to further components if the architecture needs it. ZAP does not specify this dialog but we assume that the ZAP messages can be forwarded as-is, with the addition of routing information for replies.

### ZAP is a Remote Procedure Call

ZAP uses the ZeroMQ request-reply pattern. Specifically, the handler binds a REP socket, and the server connects a REQ socket. In practice the servers may use DEALER sockets in order to send multiple requests at once, and handlers may use ROUTER sockets for the same reason. Furthermore, ZAP messages contain the request-reply address envelope so that requests can be sent along multiple intermediaries, and replies will return to the right server.

### Extensible Security

ZAP makes no assumptions about the security mechanisms we use, nor how they work. An authentication request starts with the name of a mechanism (an ASCII string), followed by credentials (an opaque binary blob whose significance is known only to the security mechanism). Additionally, the server specifies a *domain* (an ASCII string), which the handler can use to implement access controls. The authenticator replies with an OK/NOT OK response, with a user id that the server can use to identify the user to internal application code.

### ZMTP 3.0 Compatibility

ZAP is designed to support http://rfc.zeromq.org/spec:23/ZMTP implementations (i.e., provide a way to implement ZMTP 3.0 authentication) though it is not limited to this use case. This document specifies how the ZMTP NULL, PLAIN and CURVE mechanisms are implemented over ZAP.

### Message Format and Version Detection

ZAP uses a simplistic multi-frame format for its messages, where the version number is in the first frame. This is meant to be easiest to implement in arbitrary languages.

## Formal Specification

### Architecture

ZAP defines a message-based dialog between a *server* that makes authentication requests, and a *handler* that answers them. The server and handler SHALL communicate using the following socket types and transports:

* The server SHALL use a REQ or DEALER socket.
* The handler SHALL use a REP or ROUTER socket.
* The handler SHALL bind to the endpoint "<tt>inproc://zeromq.zap.01</tt>".
* The server SHALL connect to this endpoint.
* There SHALL be one handler in a process.
* There MAY be any number of servers in a process.
* The handler SHALL start before any server starts.

### Message Format

An authentication dialog SHALL consist of one request from a server to the handler, with one reply from the handler back to the server.

The request message SHALL consist of the following message frames:

* An address delimiter frame, which SHALL have a length of zero.
* The version frame, which SHALL contain the three octets "1.0".
* The *request id*, which MAY contain an opaque binary blob.
* The *domain*, which SHALL contain a string.
* The *address*, the origin network IP address.
* The *identity*, the connection Identity, if any.
* The *mechanism*, which SHALL contain a string.
* The *credentials*, which SHALL be zero or more opaque frames.

The reply message SHALL consist of the following message frames:

* An address delimiter frame, which SHALL have a length of zero.
* The version frame, which SHALL contain the three octets "1.0".
* The *request id*, which MAY contain an opaque binary blob.
* The *status code*, which SHALL contain a string.
* The *status text*, which MAY contain a string.
* The *user id*, which SHALL contain a string.
* The *metadata*, which MAY contain a blob.

The various fields have these meanings:

* request id: the meaning of this is defined by the sending server only. The reply SHALL echo the request id without modifying it.
* domain: this requests authentication within some domain. The significance of domains are an application issue and not relevant to ZAP.
* address: this provides the IP address of the client, to allow address-based filtering. It SHALL be an IPv4 dotted string, or an IPv6 canonical string representation.
* identity: this provides the Identity metadata property, if any, provided by the connection. It SHALL be a binary string no longer than 255 bytes.
* mechanism: this specifies the security mechanism to authenticate against. The mechanism SHALL NOT be empty.
* credentials: these provide the user credentials to authenticate. The number of frames needed is defined by each mechanism.
* status code: this shall be "200" to indicate success, "300" to indicate a temporary error, "400" to indicate authentication failure, and "500" to indicate an internal error (system failure).
* status text: this shall be a text that further explains the status, and MAY be empty.
* user id: this MAY provide the user identity in case of a 200 status, for use by applications. For other statuses, it SHALL be empty.
* metadata: this provides metadata to be returned to the client in case of a 200 status. The metadata SHALL use the ZMTP 3.0 format:

```
metadata = *property
property = name value
name = OCTET 1*name-char
name-char = ALPHA | DIGIT | "-" | "_" | "." | "+"
value = 4OCTET *OCTET       ; Size in network byte order
```

All string fields are at most 255 characters and are ASCII only.

### Socket Types

The ZeroMQ REQ and REP sockets manage address envelopes automatically, while the DEALER and ROUTER sockets expose these to the calling application, which must manage them explicitly.

* When using a REQ socket, the server SHALL not send nor receive an address delimiter.
* When using a REP socket, the handler SHALL not send nor receive an address delimiter.
* When using a ROUTER socket, the handler SHALL receive an extra frame indicating the routing ID of the peer that sent it the request. The handler MUST send this routing ID frame at the start of its reply message.

### Proxy Handlers

A handler MAY act as a proxy for other handlers. This is how we connect external authentication handlers using ZAP. In this case the handler will connect or bind to a <tt>tcp://</tt> endpoint, and external handlers will bind or connect to that endpoint.

In a proxy architecture the handler SHOULD forward ZAP messages with no modifications except:

* The handler MUST use a ROUTER socket for talking to servers.
* The handler MUST send the whole received message (including the first routing ID frame) on to external handlers.

More generally, a terminal (non-proxy) handler that deals with an unknown number of intermediaries between it and the servers MUST accept an address envelope consisting of N routing ID frames followed by an empty address delimiter frame. It MUST save this envelope before processing the request, and MUST send the same envelope back with the reply. This is what a REP socket does, so in general terminal handlers will use REP sockets.

### The NULL Mechanism

The NULL mechanism provides no security credentials but allows a server to filter bogus clients on the basis of IP address.

To perform filtering for the NULL mechanism:

* The mechanism SHALL be the 4 characters "NULL".
* There SHALL be no credentials frames.

### The PLAIN Mechanism

The PLAIN mechanism defines a simple username/password mechanism for ZMTP v3.0 that lets a server authenticate a client. PLAIN makes no attempt at security or confidentiality. It is intended for use on internal networks where security requirements are low.

To perform authentication for the PLAIN mechanism:

* The mechanism SHALL be the 5 characters "PLAIN".
* The credentials SHALL consist of two string frames: a username and a password.

### The CURVE Mechanism

The CURVE mechanism provides secure authentication and confidentiality for ZMTP v3.0. This mechanism implements http://rfc.zeromq.org/spec:26/CURVEZMQ security mechanism. It is intended for use on public networks where security requirements are high.

To perform authentication for the CURVE mechanism:

* The mechanism SHALL be the 5 characters "CURVE".
* The credentials SHALL consist of a 32-byte long-term public key of the peer being authenticated.

### Discovery

ZAP does not define how to discover handlers on the network. Some plausible options are hard-coded TCP endpoints or beacon discovery (as implemented by http://rfc.zeromq.org/spec:20/ZRE).

## Example Exchanges

These examples show the messages *as sent on the wire*. The messages received or sent by applications (specifically, for REQ, REP, or ROUTER sockets) will have different addressing envelopes at the start. We do not show the frame sizes or flags, only frame contents.

### Example of NULL Authentication

This shows an authentication request sent by a server to a handler, for a client using the NULL mechanism:

```
+-+
| |                 Empty delimiter frame
+-+---+
| 1.0 |             Version number
+-----++
| 0001 |            Request ID, for example "0001"
+------+
| test |            Domain, empty in this case
+------+-------+
| 192.168.55.1 |    Address
+------+-------+
| BOB  |            Identity property
+------+
| NULL |            Mechanism
+------+
```

If the server application uses a REQ socket, it SHALL NOT send the delimiter frame. If it uses a DEALER socket, it SHALL send the delimiter frame.

This shows an example reply from the handler to the server:

```
+-+
| |                 Empty delimiter frame
+-+---+
| 1.0 |             Version number
+-----++
| 0001 |            Request ID echoed from request
+-----++
| 200 |             Status code
+----++
| OK |              Status text
+----++
| joe |             User id
+-+---+
| |                 Metadata, empty
+-+
```

If the server application uses a REP socket, it SHALL NOT receive nor send the delimiter frame. If it uses a ROUTER socket, it SHALL receive and send the delimiter frame, and shall additionally receive and send an routing ID frame before the delimiter frame.

### Example of PLAIN Authentication

This shows an authentication request sent by a server to a handler, for a client using the PLAIN mechanism:

```
+-+
| |                 Empty delimiter frame
+-+---+
| 1.0 |             Version number
+-----++
| 0001 |            Request ID, for example "0001"
+------+
| test |            Domain, empty in this case
+------+-------+
| 192.168.55.1 |    Address
+-------+------+
| BOB   |           Identity property
+-------+
| PLAIN |           Mechanism
+-------+
| admin |           Username
+-------++
| secret |          Password
+--------+
```

This shows an example reply from the handler to the server:

```
+-+
| |                 Empty delimiter frame
+-+---+
| 1.0 |             Version number
+-----++
| 0001 |            Request ID echoed from request
+-----++
| 200 |             Status code
+----++
| OK |              Status text
+----++
| joe |             User id
+-+---+
| |                 Metadata, empty
+-+
```

## Reference Implementation

A minimal empty reference implementation is provided in the RFC repository at https://github.com/zeromq/rfc/blob/master/src/spec_27.c. This implementation demonstrates a server talking to a proxy handler over <tt>inproc://</tt>, talking to an external terminal handler over TCP. The terminal handler implements a PLAIN authentication mechanism.
