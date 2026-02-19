---
title: "Unbound DATA for CONNECT in HTTP/3"
abbrev: "Unbound DATA for CONNECT in HTTP/3"
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

This document defines a new HTTP/3 frame type, `UNBOUND_DATA`, and a corresponding `SETTINGS` parameter that enables endpoints to negotiate its use. When an endpoint sends an `UNBOUND_DATA` frame on a CONNECT request or response stream, it indicates that all subsequent octets on that stream are interpreted as tunneled bytes. This applies both to octets transmitted after CONNECT or extended CONNECT. The use of `UNBOUND_DATA` removes the need to encapsulate each portion of the data in `DATA` frames, reducing framing overhead and simplifying transmission of long-lived CONNECT tunnels.

--- middle

# Introduction

{{H3}} transmits message content on client-initiated bidirectional QUIC streams. On these streams, the request and response messages are carried using a sequence of HTTP/3 frames. The `DATA` frame is used to encapsulate octets of the opaque data associated with CONNECT and its extensions.

CONNECT and extended CONNECT establish long-lived bidirectional tunnels. These tunnels commonly carry transport protocols (TCP {{?CONNECT-TCP=I-D.ietf-httpbis-connect-tcp}}, UDP {{?CONNECT-UDP=RFC9298}}, IP {{?CONNECT-IP=RFC9484}}, Ethernet {{?CONNECT-ETHERNET=I-D.ietf-masque-connect-ethernet}}, {{?WebSockets=RFC9220}}, {{?WebTransport=I-D.ietf-webtrans-http3}}) that produce arbitrarily fragmented and continuous byte streams. Senders therefore generate large numbers of DATA frames whose boundaries have no semantic meaning. Although DATA frames are lightweight, each adds framing overhead and requires the sender to manage frame boundaries. For long-lived or high-volume streams, this overhead is unnecessary because the end of the QUIC stream already provides a natural message boundary.

According to {{Section 4.4 of H3}}, once HTTP/3 CONNECT tunnel is established, the stream carries opaque bytes until the QUIC FIN. CONNECT streams do not carry trailers and no additional HTTP frames are defined after tunnel establishment. Therefore, ceasing frame parsing after `UNBOUND_DATA` does not change HTTP semantics and only removes framing overhead.

This document defines a new HTTP/3 frame type, `UNBOUND_DATA`, and a corresponding `SETTINGS` parameter that endpoints use to negotiate support. Once an `UNBOUND_DATA` frame is sent on a CONNECT stream, all subsequent octets on that stream are interpreted as data. This mechanism eliminates the need to encapsulate each portion of the tunnel payload in separate DATA frames.

The goals of `UNBOUND_DATA` are:

- Reduce framing overhead for CONNECT tunnels carrying continuous byte streams.
- Simplify sender and receiver state machines by eliminating repeated DATA frame headers.
- Enable efficient transport of tunneled protocols carried over CONNECT streams.

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

The `UNBOUND_DATA` frame (type=0x2a937388) is used on CONNECT streams to indicate that all subsequent octets on the stream are interpreted as data.

## Frame Layout

~~~
UNBOUND_DATA Frame {
  Type (i) = 0x2a937388,
  Length (8) = 0,
}
~~~
{: #fig-unbound-data-frame title="HTTP/3 UNBOUND_DATA Frame"}

The `UNBOUND_DATA` frame has no payload. The Length field of the frame MUST be zero. If a nonzero length is received, the endpoint MUST treat this as a connection error of type `H3_FRAME_ERROR`.

The `UNBOUND_DATA` frame is only valid on CONNECT streams. If an endpoint receives an `UNBOUND_DATA` frame on a stream that is not a CONNECT stream, it MUST treat it as a connection error of type `H3_FRAME_UNEXPECTED`.

Similar to `DATA` frames, endpoints MUST send a `HEADERS` frame before sending an `UNBOUND_DATA` frame on a given stream. Receipt of an `UNBOUND_DATA` frame on a stream that hasn't received a `HEADERS` frame MUST be treated as a connection error of type `H3_FRAME_UNEXPECTED`.

## Semantics

Upon receiving an `UNBOUND_DATA` frame on a CONNECT stream, the receiver enters unbound mode for that stream. In unbound mode:

- All remaining octets on the stream, up to the QUIC FIN, are interpreted as data.
- No further HTTP/3 frames (including `DATA`, `HEADERS`, or any extension frames) can be received on the stream.
- The end of the data is indicated by the QUIC FIN on the stream.

Unbound mode is direction-specific: receipt of `UNBOUND_DATA` only affects the interpretation of octets received in that direction of the stream. Each endpoint can independently send `UNBOUND_DATA` to indicate unbound mode for its sending direction.

# Stream State Transitions

The use of the `UNBOUND_DATA` frame modifies the sequence of frames exchanged on request and response streams.

In normal operation, a CONNECT request or response is carried as a sequence of one or more `DATA` frames:

~~~aasvg
  New bi-direcitonal QUIC stream ---->  +------------------------+
                                        | HEADERS (headers)      |
                                        +------------------------+
                                        | [ DATA ... ]           |
                        QUIC FIN ---->  +------------------------+
~~~
{: #fig-regular-http3-state title="Regular HTTP/3 CONNECT frame sequence on bi-directional stream"}

When `UNBOUND_DATA` is used, the sender signals that all subsequent octets on the stream are data. Regular `DATA` frames MAY be sent on a stream prior to the `UNBOUND_DATA`. After the `UNBOUND_DATA` frame, the sender cannot send any further HTTP/3 frames on the stream. The end of the tunnel is signaled by the QUIC stream FIN:

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

# Security Considerations

The introduction of `UNBOUND_DATA` does not alter the security properties of HTTP/3 or QUIC. It only changes how the CONNECT payload is framed on request and response streams.

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
