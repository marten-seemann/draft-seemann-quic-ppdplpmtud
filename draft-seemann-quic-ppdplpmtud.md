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
 -
    name: Lars Eggert
    org: Mozilla
    email: lars@eggert.org
    uri: https://eggert.org/

normative:
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

QUIC endpoints commonly use 1200-byte datagrams during the handshake and only
start Path MTU Discovery afterward. This means that just-established QUIC
connections cannot immediately use larger datagrams, which is especially limiting
for MASQUE protocols and for WebTransport.
This document defines Parallel Probing DPLPMTUD (PPDPLPMTUD), which probes
several packet sizes early during the QUIC handshake, so a larger discovered size is
usable during later handshake phases and especially after handshake completion.
The same discovery process is also used for path migration.



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

PPDPLPMTUD sends parallel probes for several distinct packet sizes
without waiting for individual results. It needs no new frames or transport parameters and
operates independently in each direction. The additional traffic is justified
only when the application is expected to benefit early in the connection.

This document first describes the generic PPDPLPMTUD mechanism, followed by how it
applies to the 1-RTT handshake, the 0-RTT handshake, and when probing a new path.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the terminology of {{RFC8899}}, {{RFC9000}}, {{RFC9001}},
and {{RFC9002}}.

# The PPDPLPMTUD Mechanism
{: #mechanism}

PPDPLPMTUD is a QUIC-specific mechanism. It does **not** probe for the MTU, i.e., it does **not** determine the **maximum** transmission unit that can be used on a path. Instead, it attempts to detect a safe larger-than-minimum packet size for a given **Internet** path.

Restricting PPDPLPMTUD to Internet paths means that the maximum packet size probed for is the minimum of the common 1500-byte Ethernet MTU and the local interface MTU on the path towards the destination. Detection of larger packet sizes is explicitly out of scope. (Note that the configured socket send buffer size and, when available, the `max_udp_payload_size` transport parameter further limit this maximum packet size.)

This section describes the general mechanism; {{handshake-1rtt}},
{{handshake-0rtt}}, and {{new-path}} describe how it applies in each
situation.

## Probe Size Selection
{: #probe-size-selection}

Probe size selection is implementation-specific. Implementations are
encouraged to use information about common PMTUs and application requirements.

Equally-spaced probe sizes are a simple choice. {{handshake-1rtt}}
gives a worked example of budgeting probes within an initial congestion
window.

## Probe Construction
{: #probe-construction}

An endpoint selects several packet sizes above its current maximum and sends
an ACK-eliciting probe with a new packet number for each. Frame types and
packet number space depend on the situation; see {{handshake-1rtt}},
{{handshake-0rtt}}, and {{new-path}}.

## Maximizing Congestion Window Utilization
{: #cwnd-utilization}

The key idea of PPDPLPMTUD is to maximize use of the available congestion
window (cwnd) for sending parallel probes: whenever cwnd headroom remains
beyond what an endpoint needs for the data it is already sending, PPDPLPMTUD
uses that headroom to send a number of packet size probes.

When an endpoint receives an acknowledgment (ACK) for a flight containing probes, it
determines the largest packet size successfully received by the peer, which
becomes the maximum datagram size for the endpoint, per {{Section 14 of RFC9000}}.

## Order and Timing
{: #order-and-timing}

Within a single packet number space, an endpoint SHOULD send probes from
largest to smallest. Smaller probes then have larger packet numbers, allowing
their ACKs to expose missing larger probes. Whether other, non-probe traffic
at the current maximum packet size is sent before or after the probes
depends on the situation; see {{handshake-1rtt}} and {{handshake-0rtt}}.

All packets, including probes, consume congestion window and MUST fit within
the available congestion window. PPDPLPMTUD MAY send a set of probes as a
burst subject to {{Section 7.7 of RFC9002}}.

## Congestion Control Interaction
{: #cc-interaction}

PPDPLPMTUD does not change congestion control. Increasing the maximum datagram
size MUST NOT increase the congestion window measured in bytes. As specified by
{{RFC8899}}, isolated loss of a packet size probe SHOULD NOT cause a
congestion control reaction.

## Confirmation
{: #confirmation}

The sender records the packet size for every probe packet number. An
ACK confirms that size and all smaller sizes. A missing
ACK does not prove that a size is unsupported: PPDPLPMTUD finds a
confirmed lower bound, not the exact path MTU. If no probe is acknowledged, the
endpoint retains its previous maximum. Regular DPLPMTUD {{RFC8899}} can
continue afterward.

Confirmation also depends on the sender still being able to process an ACK
when it arrives. Only packets coalesced into the same UDP datagram are
guaranteed to be received and processed together, since a datagram is
delivered whole or not at all -- relevant once an endpoint stops processing
a packet number space it no longer needs.

## Directionality
{: #directionality}

Results are directional and path-specific. Migration, NAT rebinding, or routing
changes can invalidate them. Cached results can guide probe sizes but MUST NOT
replace confirmation on the current path; see {{new-path}}.

# PPDPLPMTUD During the 1-RTT Handshake
{: #handshake-1rtt}

During the 1-RTT handshake, each side abandons a packet number space --
discarding its keys and stopping both sending and processing there -- based
on its own progress, not on having received everything the peer sent, per
{{Section 4.9 of RFC9001}}. The handshake MUST be able to complete even if no
probe above the current maximum is ever acknowledged, and an endpoint SHOULD
send probes promptly, or send fewer of them, to avoid leaving any unsent when
keys are discarded.

An endpoint therefore sends its probes before its essential handshake data --
the ClientHello for a client, or the Certificate, CertificateVerify, and
Finished for a server -- which it sends in packets no larger than its
current maximum. New, non-retransmitted data MUST use the sender's highest
currently-available encryption level; probes SHOULD instead carry copies of
that essential CRYPTO data, which, being a retransmission, remains permitted
at a previous level, per {{Section 4.9 of RFC9001}}. Where the data does not
fit into a single probe, copies of each range SHOULD span different probe
sizes, so the peer can assemble everything it needs from whichever probes,
or trailing packets, survive.

{{Section 13.2.1 of RFC9000}} already requires an endpoint to acknowledge
ack-eliciting Initial and Handshake packets immediately. Since generating
that ACK is unavoidable once an endpoint has everything it needs to build
its own next handshake message, it SHOULD coalesce that ACK into the same
datagram as that message (see {{confirmation}}), so it covers the peer's
probes too without waiting for the rest of the flight.

The Client and Server sections below cover what differs between the two
first flights.

## Client

For example, within the 12000-byte initial congestion window defined by
{{Section 7.2 of RFC9002}}, a client can reserve 2400 bytes at the end for
its ClientHello, sent as two 1200-byte datagrams, leaving 9600 bytes for
probes sent first. With an upper probe size of 1472 bytes, it can send seven
probes of 1472, 1433, 1394, 1355, 1317, 1278, and 1239 bytes, followed by the
ClientHello. The probes consume 9488 bytes and the complete flight consumes
11888 bytes. Together with the current maximum of 1200 bytes, these sizes
provide confirmation points approximately 39 bytes apart.

The client does not yet know the server's `max_udp_payload_size`, so probes
might exceed that value. Upon receiving the transport parameter, the client
MUST cap its maximum datagram size at the smaller of that value and the
largest acknowledged probe. An ACK does not override the transport
parameter. After Retry, the client MAY repeat or omit probing.

## Server

After processing the ClientHello, the server knows the client's
`max_udp_payload_size` and MUST NOT exceed it.

By the time it builds this flight, the server already has both Handshake and
1-RTT keys, unlike the client's first flight, where Initial is the only
level available. A probe MAY instead just contain a PING frame and enough
PADDING frames to reach the selected size; since that is new data rather
than a CRYPTO retransmission, it MUST then use 1-RTT rather than Handshake.

## Address Validation
{: #address-validation}

Before address validation, every probe byte counts against the server's
anti-amplification limit defined in {{Section 8.1 of RFC9000}}. The server MUST
prioritize handshake data and send fewer probes or none when its budget is
insufficient. Client probes increase that budget.

# PPDPLPMTUD During a 0-RTT Handshake
{: #handshake-0rtt}

0-RTT and 1-RTT packets share the same packet number space, and a client
never sends a 0-RTT packet after it has sent a 1-RTT packet, per
{{Section 4.9.3 of RFC9001}}. A size confirmed by an ACK of a 0-RTT probe
therefore applies directly to subsequent 1-RTT packets.

A client SHOULD send its 0-RTT
application data in packets at the current maximum packet size to avoid size-based loss.

Any remaining cwnd can then be used to send packet-size probes.
CRYPTO frames cannot be sent in 0-RTT packets, per {{Section 12.5 of RFC9000}},
so a 0-RTT probe cannot use the redundant-CRYPTO-copy construction described for
the handshake. A probe can still contain a PING frame and enough PADDING
frames to reach the selected size. Similarly to the redundant CRYPTO copies,
the client MAY also opportunistically retransmit STREAM data in probe
packets.

## Coalesced Probes

A probe MAY be a coalesced datagram containing packets from multiple
encryption levels, as described in {{Section 12.2 of RFC9000}}. A client can
coalesce its Initial and 0-RTT packets this way. Because both are part of the
client's first flight,
they are always processed together and neither risks the key-discard hazard
described in {{handshake-1rtt}}. Coalescing an Initial or 0-RTT packet with a
1-RTT packet in the same probe datagram, once 1-RTT keys are available,
additionally hedges against loss or discard of the earlier-space packet: if
the peer acknowledges any one of the coalesced packets, the size of the whole
datagram is confirmed.

## Client

0-RTT data and probes share the initial congestion window. A few small
requests can leave room for probing, but more 0-RTT data makes probing less
useful; the client MAY then send fewer probes, omit PPDPLPMTUD, or
retransmit STREAM data as described above to avoid the tradeoff.

A client that attempts 0-RTT MUST remember the server's previously provided
`max_udp_payload_size`, per {{Section 7.4.1 of RFC9000}}.

A server MAY reject 0-RTT instead of selecting a smaller value than it
previously provided, but this check is optional, so an accepted handshake
does not guarantee the remembered value still holds; the client MUST still
cap on the actual transport parameter once received, as in the 1-RTT case.
Unlike in a 1-RTT handshake, where the client has no prior information, a
0-RTT client has an upper bound to size its probes against. If
the server rejects 0-RTT, the client falls back to the 1-RTT handshake
behavior.

## Server

Anti-amplification limits apply before address validation just as during a
1-RTT handshake (see {{address-validation}}). The server MUST prioritize
processing the ClientHello and any 0-RTT data it accepts, sending probes only
within the remaining congestion window; since the server has 1-RTT keys
available essentially as soon as it starts building its response, these
probes use 1-RTT PING and PADDING frames, for the same reason given for the
1-RTT handshake.

# PPDPLPMTUD When Probing a New Path
{: #new-path}

QUIC endpoints validate a new path, as
described in {{Section 8.2 of RFC9000}}. Before committing to migrate there,
an endpoint can test a new path's viability, including the packet sizes it
supports, using only "probing frames" ({{Section 9.1 of RFC9000}}): a
PPDPLPMTUD probe on a not-yet-validated path is a packet built from PADDING
together with a
PATH_CHALLENGE, PATH_RESPONSE, or NEW_CONNECTION_ID frame, padded to the
selected probe size.

Before the new address is validated, an endpoint sending to it MUST NOT send
more than three times the bytes it has received from that address, per
{{Section 21.1.1.1 of RFC9000}}. Any datagram containing a
PATH_CHALLENGE frame MUST already be expanded to at least the minimum QUIC
datagram size of 1200 bytes unless the anti-amplification limit does not
permit it, per {{Section 8.2.1 of RFC9000}}; PPDPLPMTUD probes extend above
that floor.

Once path validation succeeds, an endpoint MUST reset its congestion
controller and RTT estimator for that path to their initial values, per
{{Section 9.4 of RFC9000}}. PPDPLPMTUD can now run an additional probing
round the same way as during the handshake (see {{mechanism}}), no longer
confined to probing frames.

A confirmed size becomes usable once the endpoint takes the new path into
use for transmission; results from the path being replaced do not carry
over.

# Security Considerations

The security considerations of {{RFC8899}}, {{RFC9000}}, {{RFC9001}}, and
{{RFC9002}} apply.

PPDPLPMTUD adds traffic and processing during connection establishment and can
cause a short burst of queueing or loss. Implementations SHOULD use only as many
sizes as provide useful resolution and SHOULD disable probing on metered networks.
Probe sizes can also fingerprint an implementation.


# IANA Considerations

This document has no IANA actions.
