---
title: WebTransport over QUIC
abbrev: QuicTransport
docname: draft-vvv-webtransport-quic-latest
date: {DATE}
category: std

ipr: trust200902
area: Transport

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: V. Vasiliev
    name: Victor Vasiliev
    organization: Google
    email: vasilvv@google.com

normative:
  QUIC-TRANSPORT:
    title: "QUIC: A UDP-Based Multiplexed and Secure Transport"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-ietf-quic-transport-latest
    author:
      -
        ins: J. Iyengar
        name: Jana Iyengar
        org: Fastly
        role: editor
      -
        ins: M. Thomson
        name: Martin Thomson
        org: Mozilla
        role: editor
  QUIC-DATAGRAM:
    title: "An Unreliable Datagram Extension to QUIC"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-pauly-quic-datagram-latest
    author:
      -
        ins: T. Pauly
        name: Tommy Pauly
        org: Apple
      -
        ins: E. Kinnear
        name: Eric Kinnear
        org: Apple
      -
        ins: D. Schinazi
        name: David Schinazi
        org: Google
  OVERVIEW:
    title: "The WebTransport Protocol Framework"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-vvv-webtransport-overview-latest
    author:
      -
        ins: V. Vasiliev
        name: Victor Vasiliev
        organization: Google
  FETCH:
    target: https://fetch.spec.whatwg.org/
    title: "Fetch Standard"
    author:
      org: WHATWG
    date: Living Standard

--- abstract

WebTransport [OVERVIEW] is a protocol framework that enables clients constrained
by the Web security model to communicate with a remote server using a secure
multiplexed transport.  This document describes QuicTransport, a transport
protocol that uses a dedicated QUIC [QUIC-TRANSPORT] connection and provides
support for unidirectional streams, bidirectional streams and datagrams.

--- middle

# Introduction

QUIC [QUIC-TRANSPORT] is a UDP-based multiplexed secure transport.  It is the
underlying protocol for HTTP/3 {{?I-D.ietf-quic-http}}, and as such is
reasonably expected to be available in web browsers and server-side web
frameworks.  This makes it a compelling transport to base a WebTransport
protocol on.

This document defines QuicTransport, an adaptation of QUIC to WebTransport
model.  The protocol is designed to be low-overhead on the server side, meaning
that server software that already has a working QUIC implementation available
would not require a large amount of code to implement QuicTransport.  Where
possible, WebTransport concepts are mapped directly to the corresponding QUIC
concepts.

## Terminology

The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

This document follows terminology defined in Section 1.2 of [OVERVIEW].  The
diagrams describe encoding following the conventions described in Section 1.3 of
[QUIC-TRANSPORT].

# Protocol Overview

Each QuicTransport uses a single dedicated QUIC connection.  This allows the
peers to exercise a greater level of control over the way their data is being
transmitted.  However, this also means that multiple instances of QuicTransport
cannot be pooled, and thus do not benefit from sharing congestion control
context with other potentially already existing connections.  Http3Transport
[I-D.vvv-webtransport-http3] can be used in situations where such pooling is
beneficial.

When a client requests a QuicTransport to be created, the user agent establishes
a QUIC connection to the specified address.  It verifies that the the server is
a QuicTransport endpoint using ALPN, and sends a *client indication* containing
the origin of the initiating website to the server.  At that point, the
connection is ready from the client's perspective.  The server MUST wait until
the indication is received before processing any application data.

WebTransport streams are provided by creating an individual unidirectional or
bidirectional QUIC stream.  WebTransport datagrams are provided through the QUIC
datagram extension [QUIC-DATAGRAM].

# Connection Establishment  {#connection}

In order to establish a QuicTransport session, a QUIC connection must be
established.  From the client perspective, the session becomes established when
the client receives a TLS Finished message from the server.

## Identifying as QuicTransport

In order to identify itself as a WebTransport application, QuicTransport relies
on TLS Application-Layer Protocol Negotiation {{!RFC7301}}.  The user agent MUST
request the ALPN value of "wq-vvv-01" and it MUST close the connection unless
the server confirms that ALPN value.

## Client Indication

In order to verify that the client's origin is allowed to connect to the server
in question, the user agent has to communicate the origin to the server.  This
is accomplished by sending a special message, called *client indication*, on
stream 2, which is the first client-initiated unidirectional stream.

The client indication is a sequence of key-value pairs that are formatted in the
following way:

~~~~~~~~~~ drawing
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Key (16)                          ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          Length (16)                        ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Value (*)                         ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #fig-frame title="Client indication format" :}

The pair includes the following fields:

  Key:
  : Indicates the field that is being expressed.

  Length:
  : Indicates the length of the value (the length of the key and the length
    itself are not included).

  Value:
  : The value of the field, the semantics of which are determined by the key.

A FIN on the stream 2 SHALL indicate that the message is complete.  The client
MUST send the entirety of the client indication and a FIN immediately after
opening the connection.  The server MUST NOT process any application data before
receiving the entirety of the client indication.  The total length of the client
indication MUST NOT exceed 65,535 bytes.

The server MUST ignore the fields it does not recognize.  All of the fields MUST
be unique;  the server MAY close the connection if any of the keys is used more
than once.

### Origin Field

In order to allow the server to enforce the origin policy, the user agent has to
communicate the origin in the client indication.  This can be accomplished using
the "Origin" field:

  Name:
  : Origin

  Key:
  : 0x0000

  Description:
  : The origin {{!RFC6454}} of the client initiating the connection.

The user agent MUST send the Origin field.  The origin field MUST be set to the
origin of the client initiating the connection, serialized as described in
[serializing a request
origin](https://fetch.spec.whatwg.org/#serializing-a-request-origin) of [FETCH].

## 0-RTT

QuicTransport provides applications with ability to use the 0-RTT feature
described in {{!RFC8446}} and [QUIC-TRANSPORT].  0-RTT allows a client to send
data before the TLS session is fully established.  It provides a lower latency,
but has the drawback of being vulnerable to replay attacks as a result.  Since
only the application can make the decision of whether some data is safe to send
in that context, 0-RTT requires the client API to only send data over 0-RTT when
specifically requested.

0-RTT support in QuicTransport is OPTIONAL, as it is in QUIC and TLS 1.3.

# Streams

QuicTransport unidirectional and bidirectional streams are created by creating a
QUIC stream of corresponding type.  All other operations (read, write, close)
are also mapped directly to the operations as defined in [QUIC-TRANSPORT].  The
QUIC stream IDs are the stream IDs that are exposed to the application.

# Datagrams

QuicTransport uses the QUIC DATAGRAM frame [QUIC-DATAGRAM] to provide
WebTransport datagrams.  A QuicTransport endpoint MUST negotiate and support the
DATAGRAM frame.  The datagrams provided by the application are sent as-is.  The
datagram ID SHALL be absent.

The datagrams sent using QuicTransport MUST be subject to congestion control.

# QuicTransport URI Scheme {#uri}

NOTE: the URI scheme defintion in this section is provisional and subject to
change, especially the name of the scheme.

QuicTransport uses `quic-transport` URI scheme for identifying QuicTransport
servers.

The syntax definition below uses Augmented Backus-Naur Form (ABNF) {{!RFC5234}}.
The definitions of `host`, `port`, `path-abempty`, `query` and `fragment` are
adopted from {{!RFC3986}}.  The syntax of a QuicTransport URI SHALL be:

~~~~~~~~~~~~~~~
quic-transport-URI = "quic-transport:" "//"
                             host [ ":" port ]
                             path-abempty
                             [ "?" query ]
                             [ "#" fragment ]
~~~~~~~~~~~~~~~

This document does not define any semantics to `path-abemtpy`, `query` or
`fragment` portions of the URI, nor does it delegate those to the host owners.
Any QuicTransport implementation MUST ignore those until a subsequent
specification assigns semantics to those.

The `host` component MUST NOT be empty.  If the `port` component is missing, the
port SHALL be assumed to be 0.

NOTE: this effectively requires the port number to be specified.  This
specification may include an actually usable default port number in the future.
At this point, we are unable to provide such until IANA allocates one.

In order to connect to a QuicTransport server identified by a given URI, the
user agent SHALL establish a QUIC connection to the specified `host` and `port`
as described in {{connection}}.  It MUST immediately signal an error to the user
if the port value is 0.

# Transport Properties

QuicTransport supports most of WebTransport features as described in
{{properties}}.

| Property            | Support
|:--------------------|:-------------------------|
| Stream independence | Always supported         |
| Partial reliability | Always supported         |
| Pooling support     | Not supported            |
| Connection mobility | Implementation-dependent |
{: #properties title="Transport properties of QuicTransport"}

# Security Considerations

QuicTransport satisfies all of the security requirements imposed by [OVERVIEW]
on WebTransport protocols, thus providing a secure framework for client-server
communication in cases when the the client is potentially untrusted.

QuicTransport uses QUIC with TLS, and as such, provides the full range of
security properties provided by TLS, including confidentiality, integrity and
authentication of the server.

QUIC is a client-server protocol where a client cannot send data until either
the handshake is complete or a previously established session is resumed.  This
ensures that the user agent will prevent the client from sending data to network
endpoints that are not QuicTransport endpoints.  Furthermore, the QuicTransport
session can be immediately aborted by the server through a connection close or a
stateless reset, causing the user agent to stop the traffic from the client.
This provides a defense against potential denial-of-service attacks on the
network by untrusted clients.

QUIC provides a congestion control mechanism {{?I-D.ietf-quic-recovery}} that
limits the rate at which the traffic is sent.  This prevents potentially
malicious clients from overloading the network.

WebTransport requires user agents to continually verify that the server is still
interested in talking to them.  QuicTransport accomplishes that by virtue of
QUIC being an ACK-based protocol; if the client is attempting to send data, and
the server does not send any ACK frames in response, the client side of the QUIC
connection will time out.

QuicTransport prevents the WebTransport clients connecting to arbitrary non-Web
servers through the use of ALPN.  Unlike TLS over TCP, successfully ALPN
negotiation is mandatory in QUIC.  Thus, unless the server explicitly picks `wq`
as the ALPN value, the TLS handshake will fail.

QuicTransport uses a unidirectional QUIC stream to provide the server with the
origin of the client.

In order to avoid the use of QuicTransport to scan internal networks, the user
agents MUST NOT allow the clients to distinguish different connection errors
before the correct ALPN is received from the server.

Since each instance of QuicTransport opens a new connection, a malicious client
can cause resource exhaustion, both on the local system (through depleting file
descriptor space or other per-connection resources) and on a given remote
server.  Because of that, the user agegts SHOULD limit the amount of
simultaneous connections opened.  The server MAY limit the amount of connections
open by the same client.

# IANA Considerations

## ALPN Value Registration

The following entry is added to the "Application Layer Protocol Negotiation
(ALPN) Protocol IDs" registry established by {{!RFC7301}}:

The "wq-vvv-01" label identifies QUIC used as a protocol for WebTransport:

  Protocol:
  : QuicTransport

  Identification Sequence:
  : 0x77 0x71 0x2d 0x76 0x76 0x76 0x2d 0x30 0x31 ("wq-vvv-01")

  Specification:
  : This document

## Client Indication Fields Registry

IANA SHALL add a registry for "QuicTransport Client Indication Fields" registry.
Every entry in the registry SHALL include the following fields:

  Name:
  : The name of the field.

  Key:
  : The 16-bit uniquie identifier that is used on the wire.

  Description:
  : A brief description of what the parameter does.

  Reference:
  : The document that describes the parameter.

The IANA policy, as descrbied in {{!RFC8126}}, SHALL be Standards Action for
values between 0x0000 and 0x03ff; Specification Required for values between
0x0400 and 0xefff; and Private Use for values between 0xf000 and 0xffff.

## URI Scheme Registration

This document contains the request for the registration of the URI scheme
`quic-transport`.  The registration request is in accordance with {{!RFC7595}}.

  Scheme name:
  : quic-transport

  Status:
  : Permanent

  Applications/protocols that use this scheme name:
  : QuicTransport

  Contact:
  : IETF Chair <chair@ietf.org>

  Change controller:
  : IESG <iesg@ietf.org>

  Reference:
  : {{uri}} of this document.

--- back
