Stupid-Simple Messaging Protocol
================================

This document describes the Stupid-Simple Messaging Protocol, an application-level
protocol for 1:1 and 1:many messaging which aims to be a lightweight alternative
to open messaging protocols such as XMPP or STOMP.

Key design goals:
  - Text-based, for easy debugging
  - Interleave request/responses and server events on a single connection
  - Simple enough that a complete and efficient client or server can be
    written in pretty much any programming language within a few hours

Copyright Notice
----------------

    Copyright (c) 2015, Hugues Bruant
    All Rights Reserved.

    This document and translations of it may be copied and furnished to
    others, and derivative works that comment on or otherwise explain it
    or assist in its implementation may be prepared, copied, published
    and distributed, in whole or in part, without restriction of any
    kind, provided that the above copyright notice and this paragraph are
    included on all such copies and derivative works.  However, this
    document itself may not be modified in any way, such as by removing
    the copyright notice, except as needed for the purpose of developing
    Internet standards in which case the procedures for copyrights defined
    in the Internet Standards process must be followed, or as required to
    translate it into languages other than English.

    The limited permissions granted above are perpetual and will not be
    revoked by Hugues Bruant or its successors or assigns.

    This document and the information contained herein is provided on an
    "AS IS" basis and HUGUES BRUANT DISCLAIMS ALL WARRANTIES, EXPRESS OR
    IMPLIED, INCLUDING BUT NOT LIMITED TO ANY WARRANTY THAT THE USE OF
    THE INFORMATION HEREIN WILL NOT INFRINGE ANY RIGHTS OR ANY IMPLIED
    WARRANTIES OF MERCHANTABILITY OR FITNESS FOR A PARTICULAR PURPOSE.


History
-------

  - 1.1: Binary payloads and updated length bounds
  - 1.0: Initial version


Introduction
------------

SSMP supports both 1:1 (aka unicast) and 1:many (aka multicast) messaging.

Unicast messages can be addressed to any peer using the identifier supplied
upon login.

Multicast messaging uses a publish/subscribe approach, where messages are
sent to a "topic" and forwarded to every client that subscribed to the
topic in question.

A limited form of broadcasting is also allowed, wherein a peer can send a
message to all peers sharing at least one topic subscription.


Terminology
-----------

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).


Assumptions
-----------

SSMP is designed to run atop a reliable 2-way streaming transport such as TCP.


Grammar
-------

Using the augmented BNF format specified in [Section 2.1 of RFC 2616](http://www.w3.org/Protocols/rfc2616/rfc2616-sec2.html)

```
message     = ( request | response | event ) LF

request     = "LOGIN" SP id SP id [ SP payload ]
            | "CLOSE"
            | "PING"
            | "PONG"
            | forwardable

response    = code [ SP payload ]

event       = "000" SP id SP ( forwardable | "PING" | "PONG" )

forwardable = "SUBSCRIBE" SP id [ SP "PRESENCE" ]
            | "UNSUBSCRIBE" SP id
            | "UCAST" SP id SP payload
            | "MCAST" SP id SP payload
            | "BCAST" SP payload
            | compat

compat      = verb [ SP id ] [ SP payload ]

code        = 3DIGIT
verb        = 1*16UPALPHA
id          = 1*64ID
payload     = ( NUL | SOH | STX | ETX ) 2*1025ANY
            | TEXT0 *1023TEXT

ID          = UPALPHA | LOALPHA | DIGIT
            | "." | ":" | "@" | "/" | "_" | "-" | "+" | "=" | "~"
TEXT0       = <any 8-bit value, except NUL, SOH, STX, ETX, and LF>
TEXT        = <any 8-bit value, except LF>
ANY         = <any 8-bit value>
UPALPHA     = <any US-ASCII uppercase letter "A".."Z">
LOALPHA     = <any US-ASCII lowercase letter "a".."z">
DIGIT       = <any US-ASCII digit "0".."9">
SP          = <US-ASCII SP, space (32)>
LF          = <US-ASCII LF, linefeed (10)>
NUL         = <US-ASCII NUL (0)>
SOH         = <US-ASCII SOH (1)>
STX         = <US-ASCII STX (2)>
ETX         = <US-ASCII ETX (3)>
```


Binary payloads
---------------

SSMP is primarily designed with text payloads in mind but it can also accommodate
binary payloads.

A binary payload can be distinguished from a text payload from the first byte:

  - for a binary payload, it MUST be one of: NUL, SOH, STX, or ETX

  - for a text payload it MUST NOT be any of these values

The first two bytes of a binary payload describe the length of the rest of the
payload. The length is computed by interpreting these two bytes as a big-endian
integer and adding one to it. This ensures that the length bounds for binary
payloads match those of text payloads.

For instance, "Hello" can be encoded as the following binary payload:

    00 04 48 65 6C 6C 6F


Response codes
--------------

Response code values are borrowed from HTTP where appropriate.

  - `200` OK
  - `400` Bad Request
  - `401` Unauthorized
  - `404` Not Found
  - `405` Not Allowed
  - `501` Not Implemented

The special `000` code is used to distinguish server events from request/responses,
thereby allowing events to be freely interleaved with regular responses on the same
connection.


Invalid messages
----------------

Upon receiving data that does not respect the protocol grammar, a server MUST
send a `400` response and immediately close the connection.

Upon receiving data that does not respect the protocol grammar, a client MUST
immediately close the connection.


Login
-----

The first client request in any connection MUST be a `LOGIN`.

The format of a `LOGIN` request is:

    LOGIN <identifier> <scheme> [ <credential> ]

If the authentication is successful, the server MUST send a `200` response
with no payload.

If no request is received after a reasonable period of time, typically a few
seconds, the server MUST close the connection without sending any response.

If the first request is not a `LOGIN` the server MUST send a `400` response
and immediately close the connection.

If the `scheme` value is not supported or if the authentication fails for
any other reason, the server MUST send a `401` response with with a space-
separated list of supported authentication schemes as payload and immediately
close the connection.

After a successful `LOGIN` request, servers MUST reject any subsequent `LOGIN`
request on the same connection with code `405`.

### Named connections

The identifier supplied upon login can be used by other peers to send unicast
messages, as described later in this document.

Upon successful login the server MUST close any previous connection that used
the same identifier.

### Anonymous connections

Servers MAY allow login with the reserved `.` user identifier.

Servers which allow anonymous login SHOULD allow multiple such connections
simultaneously.

Anonymous connections are intended for publishers. They MAY NOT subscribe to
any topic and cannot receive unicast messages but they can publish messages to
existing topics.

### Authentication schemes

#### Client certificate

All servers that accept connection over SSL/TLS MUST allow authentication through
client certificates.

The `scheme` value for certificate authentication is `cert`.

When a connection is made with a client certificate, `LOGIN` MUST succeed for
any `identifier` matching either the Common Name or one of the Subject Alternative
Names specified in the client certificate.

To accommodate multiple connections being opened using the same certificate,
servers MAY accept identifiers consisting of a valid Common Name or Subject
Alternative Name followed by a forward slash (`/`) and a sequence of one or
more `ID` characters.

#### Shared secret

Servers MAY allow authentication through a pre-shared secret.

The `scheme` value for shared secret authentication is `secret`.

#### Open login

Servers MAY allow unauthenticated `LOGIN`.

The `scheme` value for open login is `open`.

The client SHOULD NOT include a `credential` field in an `open` login request
and the server MUST ignore its content if it is present.

Open login is subject to easy abuse and SHOULD therefore only be enabled for
debugging purposes.

#### Other

Servers MAY support other authentication schemes. The `scheme` value MUST be a
sequence of one or more `ID` characters.


Ping
----

Periodic ping messages are used to test connection liveness and prevent closure
by aggressive firewalls.

### Client-initiated

Clients SHOULD send a `PING` request after an implementation-defined period where
no server event is received, typically about 30s.

Upon reception of a `PING` request, servers MUST send a `PONG` event using the
anonymous identifier as its provenance.

    Client          Server

    PING    ---->

            <----   000 . PONG

Clients SHOULD close the connection if no `PONG` event is received during an
implementation-defined period, typically 30s, after a `PING` request was sent.

### Server-initiated

Servers SHOULD send a `PING` event after an implementation-defined period where
no client request is received, typically about 30s. The provenance MUST be the
anonymous identifier.

Upon reception of a `PING` event, clients MUST send a `PONG` message.

Servers MUST NOT send any message in response to a `PONG` message.

    Client          Server

            <----   000 . PING

    PONG    ---->

Servers SHOULD close the connection if no `PONG` message is received during an
implementation-defined period, typically 30s, after a `PING` event was sent.


Topic subscriptions
-------------------

### Subscribe to multicast topic

Opt in to receiving events for messages sent to a multicast topic.

    SUBSCRIBE <topic> [ PRESENCE ]

Any `SUBSCRIBE` request from an anonymous user MUST be rejected with code `405`.

If the caller was already subscribed to the given topic, the server MUST respond
with code `409`.

The optional `PRESENCE` flag can be used to subscribe to presence notifications,
as described later in the following section.

### Unsubscribe from multicast topic

Opt out of receiving events for messages sent to a multicast topic.

    UNSUBSCRIBE <topic>

Any `UNSUBSCRIBE` request from an anonymous user MUST be rejected with code `405`.

If the caller was not subscribed to the given topic, the server MUST respond with
code `404`.


Presence notifications
----------------------

When the `PRESENCE` flag is provided, the caller will receive an initial batch
of `SUBSCRIBE` events for all current subscribers and subsequently, `SUBSCRIBE`
and `UNSUBSCRIBE` events as topic membership changes.

The server MUST ensure that presence notifications are delivered in a safe
order. Crucially, an `UNSUBSCRIBE` event MUST NOT be re-ordered before the
corresponding `SUBSCRIBE` event. 

Upon successful subscription, the server MUST forward the message to every
client who specified the `PRESENCE` flag when subscribing to the topic.

    000 <from> SUBSCRIBE <topic> [ PRESENCE ]

Upon successful unsubscription, the server MUST forward the message to every
client who specified the `PRESENCE` flag when subscribing to the topic.

    000 <from> UNSUBSCRIBE <topic>


Messages
--------

Message delivery is:
  - in-order: two messages from the same sender to the same recipient MUST arrive
    in-order at the recipient
  - at most once: recipients MUST NOT receive duplicate messages
  - best effort with no acknowledgment: a successful response from the server
    indicates that the message was received by the server but not necessarily
    by the final recipient

### Unicast

Send message to a single peer.

    UCAST <to> <payload>

If no peer with the requested identifier is currently connected, the server
MUST send a `404` response.

Otherwise it MUST forward the message to the given peer:

    000 <from> UCAST <to> <payload>


### Multicast

Send message to all peers subscribed to a given topic.

    MCAST <topic> <payload>

The server MUST NOT send a 404 response, even if no peer has subscribed to the
given topic.

The server MUST forward the message to to every peer having subscribed to the
topic, except the sender.

    000 <from> MCAST <topic> <payload>

A client does not need to be subscribed to a topic to send messages to it.

### Broadcast

Broadcast to all peers sharing at least one topic.

    BCAST <payload>

Any `BCAST` request from an anonymous user MUST be rejected with code `405`.

The server MUST send forward the message to every peer sharing at least one
topic with the sender of the `BCAST` request.

    000 <from> BCAST <payload>

Peers that share multiple topics with the sender MUST NOT receive multiple
identical `BCAST` events.


Closing connections
-------------------

A client may cleanly close a connection by sending a `CLOSE` request.

    CLOSE


Upon receiving a `CLOSE` request, servers MUST reply with code `200` and
immediately close the connection.

Servers MUST send appropriate `UNSUBSCRIBE` events for all topics to which
the client was subscribed.

Similarly, if the server closes a connection for any reason, either mandated
by this specification or due to underlying network issues, it MUST send
appropriate `UNSUBSCRIBE` events.


Forward compatibility
---------------------

Upon reception of a request with an unrecognized `verb`, servers MUST send a
`501` response and keep the connection open.

This is intended to allow client to safely detect whether servers support
any new or optional requests that may be added in future versions of this
specification.

Upon reception of an event with an unrecognized `verb`, clients MUST immediately
close the connection.


Payload encoding considerations
-------------------------------

Servers are oblivious to the encoding of text payloads. They MUST NOT make any
assumption about character set or encoding and MUST NOT alter the contents of
the payload in any way before forwarding them to recipients.

Clients SHOULD encode text payloads as UTF-8.


Security considerations
-----------------------

Clients and servers SHOULD secure communications by connecting over TLS,
especially if a pre-shared secret is used for authentication purposes.


Known implementations
---------------------

 * [lipwig](https://github.com/aerofs/lipwig) : reference implementation (server and client)
 * [jssmp](https://github.com/aerofs/jssmp) : Java implementation (server and client)

