---
title: "Unbound DATA Frames in HTTP/3"
abbrev: "Unbound DATA Frames in HTTP/3"
category: std

docname: draft-rosomakho-httpbis-h3-unbound-data-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "HTTP"
keyword:
 - http
 - data
 - unbound
venue:
  group: "HTTP"
  type: "Working Group"
  mail: "ietf-http-wg@w3.org"
  arch: "https://lists.w3.org/Archives/Public/ietf-http-wg/"
  github: "yaroslavros/draft-httpbis-h3-unbound-data"
  latest: "https://yaroslavros.github.io/draft-httpbis-h3-unbound-data/draft-rosomakho-httpbis-h3-unbound-data.html"

author:
 -
    fullname: Yaroslav Rosomakho
    organization: Zscaler
    email: yrosomakho@zscaler.com
 -
    ins: D. Schinazi
    name: David Schinazi
    organization: Google LLC
    email: dschinazi.ietf@gmail.com

normative:
  H3:
    =: RFC9114
    display: HTTP/3

informative:

--- abstract

This document defines a new HTTP/3 frame type, `UNBOUND_DATA`, and a corresponding `SETTINGS` parameter that enables endpoints to negotiate its use. When an endpoint sends an `UNBOUND_DATA` frame on a request or response stream, it indicates that all subsequent octets on that stream are interpreted as data. This applies both to message body data and to octets transmitted after CONNECT or extended CONNECT. The use of `UNBOUND_DATA` removes the need to encapsulate each portion of the data in `DATA` frames, reducing framing overhead and simplifying transmission of long-lived or indeterminate-length payloads.

--- middle

# Introduction

{{H3}} transmits message content on client-initiated bidirectional QUIC streams. On these streams, the request and response messages are carried using a sequence of HTTP/3 frames. The `DATA` frame is used to encapsulate octets of a message body, or the opaque data associated with CONNECT and its extensions.

When the size of the body is unknown in advance, or when data is generated incrementally such as media streaming, {{?WebSockets=RFC9220}}, {{?WebTransport=I-D.ietf-webtrans-http3}}, or other tunneled protocols using CONNECT and its extensions, senders typically generate many DATA frames. Although DATA frames are lightweight, each adds framing overhead and requires the sender to manage frame boundaries. For long-lived or high-volume streams, this overhead is unnecessary because the end of the QUIC stream already provides a natural message boundary.

This document defines a new HTTP/3 frame type, `UNBOUND_DATA`, and a corresponding `SETTINGS` parameter that endpoints use to negotiate support. Once an `UNBOUND_DATA` frame is sent on a request or response stream, all subsequent octets on that stream are interpreted as data. This mechanism eliminates the need to encapsulate each portion of the body in separate DATA frames.

The goals of `UNBOUND_DATA` are:

- Reduce framing overhead for large or indeterminate-length message bodies.
- Simplify sender and receiver state machines by eliminating repeated DATA frame headers.
- Enable efficient transport of long-lived data flows such as streaming APIs, media delivery, or tunneled protocols.

The use of UNBOUND_DATA does not alter HTTP semantics, flow control, or prioritization; it is strictly a framing optimization.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Capability Negotiation

Endpoints indicate support for unbound data transmission by sending the `SETTINGS_ENABLE_UNBOUND_DATA` (0x282cf6bb) setting with a value of 1.

The valid values of the `SETTINGS_ENABLE_UNBOUND_DATA` setting are 0 and 1. If the `SETTINGS_ENABLE_UNBOUND_DATA` setting is received with a different value, the receiver MUST treat it as a connection error of type `H3_SETTINGS_ERROR`.

A value of 1 indicates that the sender of the SETTINGS frame is willing to receive `UNBOUND_DATA` frames.

Endpoints MUST NOT send an `UNBOUND_DATA` frame to a peer that has not advertised `SETTINGS_ENABLE_UNBOUND_DATA` with a value of 1. Endpoints that receive an `UNBOUND_DATA` frame without having advertised support MUST treat it as a connection error of type `H3_FRAME_UNEXPECTED`.

The `SETTINGS_ENABLE_UNBOUND_DATA` parameter is directional: each endpoint independently advertises whether it accepts receiving `UNBOUND_DATA`. An endpoint that has not indicated support cannot be assumed to understand or correctly process the frame.

# UNBOUND_DATA Frame

The `UNBOUND_DATA` frame (type=0x2a937388) is used on request or response streams to indicate that all subsequent octets on the stream are interpreted as data. This data can represent an HTTP message body or the data stream as defined in {{Section 3.1 of !HTTP-DGRAM=RFC9297}}.

## Frame Layout

~~~
UNBOUND_DATA Frame {
  Type (i) = 0x2a937388,
  Length (8) = 0,
}
~~~
{: #fig-unbound-data-frame title="HTTP/3 UNBOUND_DATA Frame"}

The `UNBOUND_DATA` frame has no payload. The Length field of the frame MUST be zero. If a nonzero length is received, the endpoint MUST treat this as a connection error of type `H3_FRAME_ERROR`.

The `UNBOUND_DATA` frame is only valid on request or response streams. If an endpoint receives an `UNBOUND_DATA` frame on a stream that isn't a client-initiated bidirectional stream, it MUST treat it as a connection error of type `H3_FRAME_UNEXPECTED`.

Similar to `DATA` frames, endpoints MUST sent a `HEADERS` frame before sending an `UNBOUND_DATA` frame on a given stream. Receipt of an `UNBOUND_DATA` frame on a stream that hasn't received a `HEADERS` frame MUST be treated as a connection error of type `H3_FRAME_UNEXPECTED`.

## Semantics

Upon receiving an `UNBOUND_DATA` frame on a request or response stream, the receiver enters unbound mode for that stream. In unbound mode:

- All remaining octets on the stream, up to the QUIC FIN, are interpreted as data.
- No further HTTP/3 frames (including `DATA`, `HEADERS`, or any extension frames) can be received on the stream.
- The end of the data is indicated by the QUIC FIN on the stream.
- If a `Content-Length` header was included, the recipient needs to ensure that the combined length of all received data (inside `DATA` frames and after the `UNBOUND_DATA` frame) matches the content length from the header.

# Stream State Transitions

The use of the `UNBOUND_DATA` frame modifies the sequence of frames exchanged on request and response streams.

In normal operation, a request or response body is carried as a sequence of one or more `DATA` frames, followed optionally by a `HEADERS` frame containing trailers:

~~~aasvg
  New bi-direcitonal QUIC stream ---->  +------------------------+
                                        | HEADERS (headers)      |
                                        +------------------------+
                                        | [ DATA ... ]           |
                                        +------------------------+
                                        | [ HEADERS (trailers) ] |
                        QUIC FIN ---->  +------------------------+
~~~
{: #fig-regular-http3-state title="Regular HTTP/3 Frame sequence on bi-directional stream"}

When `UNBOUND_DATA` is used, the sender signals that all subsequent octets on the stream are data. Regular `DATA` frames MAY be sent on a stream prior to the `UNBOUND_DATA`. After the `UNBOUND_DATA` frame, the sender cannot send any further HTTP/3 frames on the stream. The end of the body is signaled by the QUIC stream FIN:

~~~aasvg
  New bi-directional QUIC stream ---->  +------------------------+
                                        | HEADERS (headers)      |
                                        +------------------------+
                                        | [ DATA ... ]           |
                                        +------------------------+
                                        | UNBOUND_DATA           |
                                        +------------------------+
                                        | Raw octets (data only) |
                        QUIC FIN ---->  +------------------------+
~~~
{: #fig-regular-http3-unbound-state title="HTTP/3 Frame sequence with UNBOUND_DATA"}

Since the recipient of an `UNBOUND_DATA` will no longer parse frame types on the stream after its receipt, it is not possible to send other frames after the `UNBOUND_DATA`. If that is required, for example if the sender wishes to send trailers, then the `UNBOUND_DATA` frame cannot be used.

# Security Considerations

The introduction of `UNBOUND_DATA` does not alter the security properties of HTTP/3 or QUIC. It only changes how message bodies or tunneled octets are framed on request and response streams.

# IANA Considerations

## HTTP/3 Setting

This specification registers the following entry in the "HTTP/3 Settings" registry defined in {{H3}}:

- Code: 0x282cf6bb
- Setting Name: SETTINGS_ENABLE_UNBOUND_DATA
- Default: 0
- Status: provisional (permanent if this document is approved)
- Reference: This document
- Change Controller: Yaroslav Rosomakho (IETF if this document is approved)
- Contact: yrosomakho@zscaler.com (HTTP_WG; HTTP working group; ietf-http-wg@w3.org if this document is approved)
- Notes: None

## HTTP/3 Frame Type

This specification registers the following entry in the "HTTP/3 Frame Types" registry defined in {{H3}}:

- Value: 0x2a937388
- Frame Type: UNBOUND_DATA
- Status: provisional (permanent if this document is approved)
- Reference: This document
- Change Controller: Yaroslav Rosomakho (IETF if this document is approved)
- Contact: yrosomakho@zscaler.com (HTTP_WG; HTTP working group; ietf-http-wg@w3.org if this document is approved)
- Notes: None

--- back

# Acknowledgments
{:numbered="false"}

This specification originated from discussions with Christian Huitema and Alan Frindell, whose ideas and feedback helped shape the approach described in this document. The authors also thank Lucas Pardue for providing valuable initial feedback.
