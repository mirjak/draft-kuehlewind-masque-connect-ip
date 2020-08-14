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


informative:
    I-D.schinazi-masque-connect-udp:
   
--- abstract

This draft  specifies the CONNECT-IP method to proxy IP traffic. A client connects
to a proxy server by initiating a HTTP/3 connection. The proxy server then forwards
payload of one HTTP datagram flow to a target server by adding an IP header to
each datagram received.


--- middle


# Introduction

This document specifies the CONNECT-IP method for IP 
{{RFC0791}} {{RFC8200}} flows when they are proxied according to the
MASQUE proposal over HTTP/3.

## Definitions

  * UDP Flow: A sequence of UDP packets sharing a 5-tuple.
  
  * ECN: Explicit Congestion Notification {{RFC3168}}.
  
  * DSCP: Differentiated Service Code Point {{RFC2474}}.
  
  * Proxy: This document uses proxy as synonym for the MASQUE Server or an HTTP
    proxy, depending on context.

  * Client: The endpoint initiating a MASQUE tunnel and UDP/IP relaying with the
    proxy.

  * Target: A remote endpoint the client wishes to establish bi-directional 
    communication with. 
    
  * UDP proxying: A proxy forwarding data to a target over an UDP
    "connection". Data is decapsulate at the proxy and amended by a UDP and IP
    header before forwarding to the target. Datagram boundaries need to be
    preserved or signalled between the client and proxy.
    
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
i.e. Client, Proxy, and Target, that are discussed below. We use the name target
for the node or endpoint the client intends to communicate with via the proxy.
There are also two network paths. Path #1 is the client to proxy path, where the
MASQUE protocol will be used over an HTTP/3 session, usually over QUIC, to
tunnel IP/UDP flow(s). Path #2 is the path between the Proxy and the Target.

The client will use the proxy's service address to establish a transport
connection on which to communicate with the proxy using the MASQUE protocol. The
MASQUE protocol will be used to establish the relaying of a IP/UDP flow from the
client using as the source address the proxy's external address and sending to
the target address. In addition, after establishment, the reverse is also
configured on the proxy; IP/UDP packets sent from the target address to the
proxy's external address will be relayed to the client.

# The CONNECT-IP method {#connect-ip-method}

This method defines a new HTTP/3 {{!I-D.ietf-quic-http}} method CONNECT-IP 
to convert streams into tunnels to a forwarding 
proxy. Each stream can be used separately to establish forwarding of one connection 
to potentially different remote hosts. Other than the HTTP CONNECT method, CONNECT-IP 
however does not request the forwarding proxy to establish an TCP connection to the remote 
target host. Instead the the end-to-end tunnel payload will be forwarded right on top of the IP
layer, meaning the forwarding proxy has to identify messages boundaries and then adds an IP
header to the message before forwarding (see section {{proxy}}).

This document specifies CONNECT-IP only for HTTP/3 following the same semantics as the
CONNECT method. As such a CONNECT-IP request MUST be constructed as follows:

*  The ":method" pseudo-header field is set to "CONNECT-IP"

*  The ":scheme" and ":path" pseudo-header fields are omitted

*  The ":authority" pseudo-header field contains the host and port to
   connect to (equivalent to the authority-form of the request-target
   of CONNECT requests; see Section 3.2.3 of {{!I-D.ietf-httpbis-messaging}})

A CONNECT request that does not conform
to these restrictions is malformed; see Section 4.1.3 of {{!I-D.ietf-quic-http}}.

The forwarding stays active as long as the respective stream is open. Forwarding can be either
realised by sending data on that stream together with an indication of message length or use of 
HTTP/3 datagrams {{!I-D.schinazi-quic-h3-datagram}} where the payload of one frame is mapped
to one message. Datagrams are mapped to a forwarding flow based on the Datagram-flow-ID
that is carried in both the HTTP/3 datagram itself as well as in the Datagram-Flow-Id Header
of the CONNECT-IP request itself as already defined in {{I-D.schinazi-masque-connect-udp}}.

## Client Behavior {#client}



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
