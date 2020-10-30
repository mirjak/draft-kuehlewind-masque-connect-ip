---
docname: draft-kuehlewind-masque-connect-ip-latest
title: The CONNECT-IP method for proxying IP traffic
abbrev: CONNECT-IP method
ipr: trust200902
category: std
date: {DATE}
area: TSV
workgroup: MASQUE

stand_alone: yes
pi: [toc, sortrefs, symrefs]


author:
  -
   ins: M. Kuehlewind
   name: Mirja Kuehlewind
   org: Ericsson
   email: mirja.kuehlewind@ericsson.com

  -
    ins: M. Westerlund
    name: Magnus Westerlund
    org: Ericsson
    email: magnus.westerlund@ericsson.com

  -
    ins: M. Ihlar
    name: Marcus Ihlar
    org: Ericsson
    email: marcus.ihlar@ericsson.com

  -
    ins: Z. Sarker
    name: Zaheduzzaman Sarker
    org: Ericsson
    email: zaheduzzaman.sarker@ericsson.com


normative:
    RFC0791:
    RFC2119:
    RFC3168:
    RFC7159:
    RFC8174:
    RFC8200:
    I-D.ietf-quic-http:
    I-D.ietf-quic-datagram:
    I-D.ietf-httpbis-messaging:
    I-D.schinazi-quic-h3-datagram:
    I-D.schinazi-masque-connect-udp:
    I-D.ietf-httpbis-header-structure:

informative:
   RFC792:
   RFC4443:


--- abstract

This draft specifies a new HTTP/3 method CONNECT-IP to proxy IP traffic. 
CONNECT-IP can be used to convert a QUIC stream into a tunnel or initialise an HTTP
datagram flow to a forwarding proxy. Each stream or HTTP datagram flow can be used
separately to establish forwarding of a connection to potentially different
remote hosts. To request forwarding, a client connects to a proxy server by
initiating a HTTP/3 connection and sends a CONNECT-IP indicating the address of
the target server. The proxy server then forwards payload received on that
stream or in an HTTP datagram with a certain flow ID to the target server after adding an
IP header to each frame received.

--- middle


# Introduction

This document specifies the CONNECT-IP method for IP {{RFC0791}} {{RFC8200}}
flows when they are proxied according to the MASQUE proposal over HTTP/3. 

The approach taken in this paper does not send the IP header as part of the payload
between the client and proxy in order to reduce overhead. The target IP address
in provided by the client as part of the CONNCT-IP request. The sources address
is selected by the proxy as further discussed below. Other information that might
be needed to construct the IP header or to inform the client about information
from received IP packets can be signalled separately.

This proposal is based on the analysis provided in  {{?I-D.westerlund-masque-transport-issues}}
indicating that most information in the IP header can or even should be provided by the proxy
as the IP communication endpoint without input needed from the client. The only information
identified that require client interaction are ECN {{RFC3168}} and ICMP {{RFC792}} {{RFC4443}}
handling. This document proposes an event-based handling for both, which may not provide unambiguous
mapping to one specific IP packet that triggered the event but trades this off for lower overhead.
DCSP {{RFC2474}} handling is considered as mainly used for local signalling and as such the proxy can
handle it independently of any client input. However, as use of DSCP could be extended in future,
the signal mechanism in MASQUE must be flexible enough to accommodate this or other future use
cases based on potentially new IPv6 extension header or destination header options {{RFC8200}}.

## Definitions

  * Proxy: This document uses proxy as synonym for the MASQUE Server or an HTTP
    proxy, depending on context.

  * Client: The endpoint initiating a MASQUE tunnel and IP relaying with the
    proxy.

  * Target host: A remote endpoint the client wishes to establish bi-directional
    communication with via tunnelling over the proxy.

  * IP proxying: A proxy forwarding data to a target over an IP
    "connection". Data is decapsulate at the proxy and amended by a IP header
    before forwarding to the target. Packet boundaries need to be preserved or
    signalled between the client and proxy.

~~~
Address = IP address + UDP port

                     Target Address --+
                                       \
+--------+           +--------+         \ +--------+
|        |  Path #1  |        | Path #2  V|        |
| Client |<--------->|  Proxy |<--------->| Target |
|        |          ^|        |^          |        |
+--------+         / +--------+ \         +--------+
                  /              \
                 /                +-- Proxy's external address
                /
               +-- Proxy's service address
~~~
{: #fig-node-model title="The nodes and their addresses"}

{{fig-node-model}} provides an overview figure of the involved nodes,
i.e. client, proxy, and target host. There are also two network paths.  Path #1
is the client to proxy path, where IP proxying is provided over an HTTP/3
session, usually over QUIC, to tunnel IP flow(s).  Path #2 is the path between
the proxy and the target.

The client will use the proxy's service address to establish a transport
connection on which to request IP proxying using HTTP/3 CONNECT-IP.  The proxy
will then relay the client's IP flows to the target host.  The IP header from
the proxy to the target carries the proxy's external address as source address
and the target's address as destination address.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when,
and only when, they appear in all capitals, as shown here.

# The CONNECT-IP method {#connect-ip-method}

This document defines a new HTTP/3 {{!I-D.ietf-quic-http}} method CONNECT-IP to
convert streams into tunnels or initialise HTTP datagram flows to a forwarding proxy. 
Each stream can be used separately to establish forwarding of one connection to potentially 
different remote hosts. Other than the HTTP CONNECT method, CONNECT-IP however does not
request the forwarding proxy to establish an TCP connection to the remote target
host. Instead the tunnel payload will be forwarded right on top of the IP layer,
meaning the forwarding proxy has to identify messages boundaries and then adds
an IP header to each message before forwarding (see section {{server}}).

This document specifies CONNECT-IP only for HTTP/3 following the same semantics
as the CONNECT method. As such a CONNECT-IP request MUST be constructed as
follows:

*  The ":method" pseudo-header field is set to "CONNECT-IP"

*  The ":scheme" and ":path" pseudo-header fields are omitted

*  The ":authority" pseudo-header field contains the host and port to
   connect to (equivalent to the authority-form of the request-target
   of CONNECT requests; see Section 3.2.3 of {{!I-D.ietf-httpbis-messaging}})

A CONNECT request that does not conform to these restrictions is malformed; see
Section 4.1.3 of {{!I-D.ietf-quic-http}}.

The forwarding stays active as long as the respective stream is open. Forwarding
can be either be realised by sending data on that stream together with an
indication of message length (see {{stream}}) or use of HTTP/3 datagrams
{{!I-D.schinazi-quic-h3-datagram}} where the payload of one frame is mapped to
one message (see {{datagram}}). 

## Stream-based mode {#stream}

Once the CONNECT-IP method has completed, only DATA frames are permitted
to be sent on that stream. Extension frames MAY be used if
specifically permitted by the definition of the extension.  Receipt
of any other known frame type MUST be treated as a connection error
of type H3_FRAME_UNEXPECTED.

Each HTTP DATA frames MUST to contain the payload of one IP
packet.

Stream based mode provides in-order and reliable delivery but may 
introduce Head of Line (HoL) Blocking if independent messages are
send in the IP payload.


## Datagram-based mode {#datagram}

The client can, in addition to stream mode, request support of
datagram mode using HTTP/3 datagram
{!H3DGRAM=I-D.schinazi-quic-h3-datagram}} to forward IP payload.

To request datagram support the client adds an Datagram-Flow-Id Header 
to the CONNECT-IP request as specified for
CONNECT-UDP in {{I-D.schinazi-masque-connect-udp}}.
Datagram mode MUST only be requested when the QUIC datagram extension
{!QUICDGRAM=I-D.ietf-quic-datagram}} was successfully negotiated during
the QUIC handshake.

Datagram mode provides un-order and unreliable delivery. In theory both, stream
as well as datagram mode, can be used in parallel, however, for most transmission
is is expected to only use one.

## Conn-ID Header for CONNECT-IP

This document further defines a new header field to be used with CONNECT-IP "Conn-ID".
The Conn-ID HTTP header field indicates the value, offset, and length of a field in the
IP payload that can be used by the MASQUE as a connection identifier in addition to the 
IP address tuple when multiple connections are proxied to the same target server.

Conn-ID is a Item Structured Header {{!STRUCT-HDR=I-D.ietf-httpbis-header-structure}}. 
Its value MUST be a Byte Sequence. Its ABNF is:

~~~
  Conn-ID = sf-binary
~~~

The following parameters are defined:
* A parameter whose name is "offset", and whose value is an Integer indicating the offset of the identifier field starting from the beginning of a datagram or HTTP frame on the forwarding stream.
* A parameter whose name is length, and whose value is an Integer indication the length of the identifier field starting from the offset.

Both parameter MUST be present and the header MUST be ignored if these parameter are not present.

This function can be used to e.g. indicate the source port field in the IP payload when containing an TCP packet.

# Client behavior {#client}

To request IP proxying, the client sends a CONNECT-IP request to the forwarding proxy
indicating the target host and port in the ":authority" pseudo-header
field. The host portion is either an IP literal encapsulated within square brackets,
an IPv4 address in dotted-decimal form, or a registered name.  Different than
for the TCP-based CONNECT, CONNECT-IP does not trigger a connection
establishment process from the proxy to the target host. Therefore, the client
does not need to wait for an HTTP response in order to send forwarding data.

Forwarding data can either be send directly on the same HTTP stream as the
CONNECT-IP request. In this case the Content-Length header is used to indicate
the length of the first message of the forwarding data. Or an HTTP datagram
encapsulated in a QUIC datagram can be send in the same QUIC packet (see below).
In this case the CONNECT-IP request MUST indicate the datagram flow ID in the
Datagram-Flow-Id Header.

QUESTION: datagram flow ID are allocated by a flow id allocation service at the server in {{!I-D.schinazi-quic-h3-datagram}}. However, with CONNECT-IP you can always send your first message directly on the same stream right after the CONNECT-IP request and sever could provide you a flow ID together with a "2xx" response to the CONNECT-IP request. Wouldn't that be easier and faster?


# MASQUE server behavior {#server}

A MASQUE server that receives an IP-CONNECT request, opens a raw socket 
and creates state to map an connection identifier, which might be a tuple,
to a target IP address. Once this is successfully established, the
proxy sends a HEADERS frame containing a 2xx series status code to
the client. To indicate support of datagram mode, if requested by the client,
the MASQUE server reflects the Datagram-Flow-Id Header from the client's
request on the HTTP response.

All DATA frames received on that stream as well as all HTPP/s datagrams 
with the specified Datagram-flow-ID are forwarded to the target server by
adding an IP header and sending the packet on the respective raw socket.

IP packets received from the target server must be mapped to an active forwarding
connection and it payload is then respectively forwarded in an DATA frame or HTTP datagram
to the client. The masque server should use the same forwarding mode as used by the client.
If both modes, datagram and stream based, are used, it is recommended to the same mode
as most recently used by the client or datagram mode as default. Alternatively, the client
might indicate a preference in the configuration file.

## Error handling

TBD (e.g. out of IP address, conn-id collision)

## IP address selection and NAT

As MASQUE server adds the IP header when sending the IP payload towards the target
server, it also select an source IP address from its pool of IP address that are routed to
the MASQUE server.

If no additional information about a payload field that can be used as an identifier based on 
Conn-ID header is provided by the client, the masque server uses the source/destination address
2-tuple in order to map an incoming IP packet to an active forwarding connection. As such
the MASQUE proxy MUST select a source IP address that leads to a unique tuple. The same IP
address can be used for different client when those client connect to different target servers,
however, this also means that potentially multiple IP address are used for the same client when
multiple connection to one target server are needed. This can be problematic if the source address
is used by the target server as an identifier.

If the Conn-D header is provided the MASQUE server should use that field as an connection identifier
together with source and destination address, as a 3-tuple. In this case it is recommended to use
a stable IP address for each client, while the same IP address might still be used for multiple clients,
if not indicated differently by the client in the configuration file. Note that if the same IP address is used for 
multiple clients, this can still lead to an identifier collision and the IP-CONNECT request MUST be reject if such 
a collision is detect.


# MASQUE signaling

One stream of the underlying QUIC connection is used as a signalling channel between
the client and proxy. Both the client and the masque server can send or request
an JSON {{RFC7159}} configuration file by sending an HTTP POST or GET to
"/.well-known/masque/config". Further the masque server can PUSH status updates
about certain forwarding streams or datagram flows, e.g. contain ECN {{RFC3168}} counters
or the outside facing IP address used for this connection, to
"/.well-known/masque/\<id\>".

Note: Alternative approach would be to use HTTP headers with IP-CONNECT for 
initial negotiation and new HTTP frame format(s) to provide per-packet information
(e.g ECN) or event-based information (e.g. ICMP).


## Config file

TBD (indicate IP address handling, forwarding mode preference, ...)


## ECN

ECN requires coordination with the e2e communication points as it should only be used
if the endpoints are also capable and willing to signal congestion notifications to the other 
end and react accordingly if a congestion notification is received. In addition, the proxy needs
to inform the client of a congestion notification (IP CE codepoint) was observed in any IP header
of a received packet from the target server. This can be realised that maintaining an CE counter
and send an updated JSON stream file if the counter changes.

Further, client must indicate to the proxy for each forwarding flow/stream if the 
ECT(0) or ECT(1) codepoint should be set. The client can update this during the lifetime
of a forwarding connection, however, there is no guarantee which packet will be forwarded 
with the updated information or the old information as QUIC datagrams may be delivered out of
order. If the IP payload is e.g. carrying TCP, today, ECN is only used after the handshake. But if not all
data packets after the handshake are immediately ECT marked, this should not have a huge impact.

It may be needed for the endpoint to validate ECN usage on the path. In this case validation can either
be done by the proxy independently or the proxy has to provide not only the number or received observed 
CE markings but also the number of sent and other received markings. This need further discussion.

## ICMP handling

TBD

# Examples

# Security considerations

This document does currently not discuss risk that are generic to the MASQUE approach.

Any CONNECT-IP specific risk need further consideration in further, especially when the 
handling of IP functions is further defined. 

# IANA considerations {#iana}

## HTTP Method {#iana-method}

This document (if published as RFC) registers "CONNECT-IP" in the HTTP Method Registry
maintained at <[](https://www.iana.org/assignments/http-methods)>.

~~~
  +--------------+------+------------+---------------+
  | Method Name  | Safe | Idempotent |   Reference   |
  +--------------+------+------------+---------------+
  | CONNECT-QUIC |  no  |     no     | This document |
  +--------------+------+------------+---------------+
~~~

## HTTP Header {#iana-header}

This document (if published as RFC) registers the "Conn-Id" header in the 
"Permanent Message Header Field Names" registry maintained at
<[](https://www.iana.org/assignments/message-headers)>.

~~~
  +-------------------+----------+--------+---------------+
  | Header Field Name | Protocol | Status |   Reference   |
  +-------------------+----------+--------+---------------+
  | Conn-Id           |   http   |  exp   | This document |
  +-------------------+----------+--------+---------------+
~~~

# Acknowledgments {#acknowledgments}
{:numbered="false"}
