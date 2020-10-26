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
    RFC8174:
    RFC8200:
    I-D.ietf-quic-http:
    I-D.ietf-httpbis-messaging:
    I-D.schinazi-quic-h3-datagram:

informative:
    I-D.schinazi-masque-connect-udp:

--- abstract

This draft specifies ta new HTTP/3 methode CONNECT-IP to proxy IP traffic. 
CONNECT-IP can be used to convert a QUIC stream into tunnels or intialize an HTTP
datagram flow to a forwarding proxy. Each stream or HTTP datagram flow can be used
separately to establish forwarding of a connection to potentially different
remote hosts. To request forwarding, a client connects to a proxy server by
initiating a HTTP/3 connection and sends a CONNECT-IP indicating the address of
the traget server. The proxy server then forwards payload received on that
stream or of an HTTP datagram flow with a certain flow ID to the target server by adding an
IP header to each frame received.

--- middle


# Introduction

This document specifies the CONNECT-IP method for IP {{RFC0791}} {{RFC8200}}
flows when they are proxied according to the MASQUE proposal over HTTP/3.

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
convert streams into tunnels to a forwarding proxy. Each stream can be used
separately to establish forwarding of one connection to potentially different
remote hosts. Other than the HTTP CONNECT method, CONNECT-IP however does not
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
indication of message length or use of HTTP/3 datagrams
{{!I-D.schinazi-quic-h3-datagram}} where the payload of one frame is mapped to
one message. Datagrams are mapped to a forwarding flow based on the
Datagram-flow-ID that is carried in both the HTTP/3 datagram itself as well as
in the Datagram-Flow-Id Header of the CONNECT-IP request as specified for
CONNECT-UDP in {{I-D.schinazi-masque-connect-udp}}.

## Client Behavior {#client}

To request IP proxying, a send a CONNECT-IP request to the forwarding proxy
indicating the target host and port in the ":authority" pseudo-header
field. Host portion is either an IP literal encapsulated within square brackets,
an IPv4 address in dotted-decimal form, or a registered name.  Different that
for the TCP-based CONNECT, CONNECT-IP does not trigger a connection
establishment process from the proxy to the target host. Therefore, the client
does not need to wait for an HTTP response in order to send forwarding data.

Forwarding data can either be send directly on the same HTTP stream as the
CONNECT-IP request. In this case the Content-Length header is used to indicate
the length of the first message of the forwarding data. An HTTP datagram
encapsulated in a QUIC datagram can be send in the same QUIC packet. In this
case the CONNECT-IP request MUST indicate the datagram flow ID in the
Datagram-Flow-Id Header.

TODO: how to know the length of follow up message on the same stream. And also
note the different properties of streams and datagrams, e.g regarding ordering
and reliability. Also discuss if both stream data and datagram data can be used
with the same forwaridng request. Is that needed ? Is there a use case for that?

## Proxy Behavior {#server}


## Examples

# Security Considerations

# IANA Considerations {#iana}

## HTTP Method {#iana-method}

This document registers "CONNECT-IP" in the HTTP Method Registry
<[](https://www.iana.org/assignments/http-methods)>.

~~~
  +--------------+------+------------+---------------+
  | Method Name  | Safe | Idempotent |   Reference   |
  +--------------+------+------------+---------------+
  | CONNECT-QUIC |  no  |     no     | This document |
  +--------------+------+------------+---------------+
~~~

# Acknowledgments {#acknowledgments}
{:numbered="false"}
