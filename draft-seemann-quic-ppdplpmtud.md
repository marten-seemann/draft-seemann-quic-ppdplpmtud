---
title: "Parallel Probing DPLPMTUD for QUIC"
abbrev: "Parallel Probing DPLPMTUD for QUIC"
category: info

docname: draft-seemann-quic-ppdplpmtud-latest

ipr: trust200902
area: "Web and Internet Transport"
workgroup: "QUIC"
keyword: Internet-Draft
venue:
  group: "QUIC"
  type: "Working Group"
  mail: "quic@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/quic/"
  github: "marten-seemann/draft-seemann-quic-ppdplpmtud"
  latest: "https://marten-seemann.github.io/draft-seemann-quic-ppdplpmtud/draft-seemann-quic-ppdplpmtud.html"

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: M. Seemann
    name: Marten Seemann
    email: martenseemann@gmail.com

normative:
  RFC2119:
  RFC8174:
  RFC8899:
  RFC9000:
  RFC9001:
  RFC9002:

informative:
  RFC9221:
  RFC9298:
  RFC9484:
  I-D.ietf-masque-connect-ethernet:
  I-D.ietf-webtrans-http3:

...

--- abstract

QUIC endpoints commonly use 1200-byte datagrams until discovering a larger path
limit after the handshake. This delays larger datagrams for MASQUE protocols
such as CONNECT-UDP, CONNECT-IP, and CONNECT-ETHERNET, and for WebTransport.
This document defines Parallel Probing DPLPMTUD (PPDPLPMTUD), which probes
several sizes during the QUIC handshake so the largest confirmed size is
available when the handshake completes.


--- middle

# Introduction

QUIC endpoints commonly limit datagrams to 1200 bytes until Datagram
Packetization Layer Path MTU Discovery (DPLPMTUD) {{RFC8899}} confirms a larger
size. Sequential probing after the handshake {{RFC9000}} can take several round
trips.

The delay is particularly undesirable for CONNECT-UDP {{RFC9298}}, CONNECT-IP
{{RFC9484}}, and CONNECT-ETHERNET {{I-D.ietf-masque-connect-ethernet}}.
Encapsulation can make 1200-byte QUIC datagrams insufficient, while discovering
an outer tunnel's MTU can delay Path MTU Discovery by an inner transport.
WebTransport over HTTP/3 {{I-D.ietf-webtrans-http3}} benefits similarly when
using QUIC DATAGRAM frames {{RFC9221}}.

PPDPLPMTUD sends probes of several sizes during the handshake without waiting
for individual results. It needs no new frames or transport parameters and
operates independently in each direction. The additional traffic is justified
only when the application is expected to benefit early in the connection.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the terminology of {{RFC8899}}, {{RFC9000}}, and
{{RFC9002}}. Datagram size means UDP payload size.


# Parallel Probing

## Probe Construction

An endpoint selects several datagram sizes above its current maximum and sends
an ack-eliciting probe with a new packet number for each. A probe can contain a
PING frame and enough PADDING frames to reach the selected size. It MAY instead
carry copies of CRYPTO data from the regular handshake flight. The handshake
MUST complete without receiving any probe above the current maximum.


## Order and Timing

The endpoint sends regular handshake packets at its current maximum before the
probes. Within each packet number space, it SHOULD send probes from largest to
smallest. Smaller probes then have larger packet numbers, allowing their
acknowledgments to expose missing larger probes to packet-threshold loss
detection.

All packets, including probes, consume congestion window and MUST fit within the
available congestion window. PPDPLPMTUD MAY send the complete flight as a burst
subject to Section 7.7 of {{RFC9002}}. An endpoint SHOULD send the sequence
over substantially less than the initial RTT, or send fewer probes, to avoid
leaving probes unsent when packet protection keys are discarded.

PPDPLPMTUD does not change congestion control. Increasing the maximum datagram
size MUST NOT increase the congestion window measured in bytes. As specified by
{{RFC8899}} and {{RFC9000}}, isolated loss of a PMTU probe SHOULD NOT cause a
congestion-control reaction. An endpoint SHOULD send at most one parallel
sequence in each direction during connection establishment.


## Confirmation

The sender records the datagram size for every probe packet number. An
acknowledgment confirms that size and all smaller sizes. A missing
acknowledgment does not prove that a size is unsupported: PPDPLPMTUD finds a
confirmed lower bound, not the exact Path MTU. If no probe is acknowledged, the
endpoint retains its previous maximum. Regular DPLPMTUD can continue after the
handshake.


## Coalesced Probes

A probe MAY be a coalesced datagram containing packets from multiple packet
number spaces, as described in Section 12.2 of {{RFC9000}}. A client can
coalesce Initial and 0-RTT packets; a server can coalesce Initial and Handshake
packets, optionally followed by a 1-RTT packet. An acknowledgment of any
constituent packet confirms the datagram size. This can preserve the result when
another packet number space is discarded and improve handshake progress under
loss.


# Endpoint Behavior

## Client

A client sends all ClientHello data in Initial packets no larger than its
current maximum before sending probes. It does not yet know the server's
`max_udp_payload_size`, so probes might exceed that value. Upon receiving the
transport parameter, the client MUST cap its maximum datagram size at the
smaller of that value and the largest acknowledged probe. An acknowledgment does
not override the transport parameter. After Retry, the client MAY repeat or omit
probing.

0-RTT data and probes share the initial congestion window. A few small requests
can leave room for probing, but more 0-RTT data makes probing less useful; the
client then sends fewer probes or omits PPDPLPMTUD.


## Server

After processing the ClientHello, the server knows the client's
`max_udp_payload_size` and MUST NOT exceed it. The server sends the regular
handshake flight first, then probes within the remaining congestion window. If
probes carry redundant CRYPTO data, copies of each range SHOULD span different
probe sizes. When 1-RTT keys are available, new PING frames use 1-RTT; Initial
and Handshake packets can carry retransmitted CRYPTO data, acknowledgments, and
padding as permitted by {{RFC9001}}.


## Address Validation

Before address validation, every probe byte counts against the server's
anti-amplification limit. The server MUST prioritize handshake data and send
fewer probes or none when its budget is insufficient. Client probes increase
that budget but reveal nothing about the server-to-client Path MTU.


# Path Changes

Results are directional and path-specific. Migration, NAT rebinding, or routing
changes can invalidate them. Cached results can guide probe sizes but do not
replace confirmation on the current path.


# Security Considerations

The security considerations of {{RFC8899}}, {{RFC9000}}, {{RFC9001}}, and
{{RFC9002}} apply.

PPDPLPMTUD adds traffic and processing during connection establishment and can
cause a short burst of queueing or loss. Implementations SHOULD use only as many
sizes as provide useful resolution and can disable probing on metered networks.
Probe sizes can also fingerprint an implementation.


# IANA Considerations

This document has no IANA actions.
