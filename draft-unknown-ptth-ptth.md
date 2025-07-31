---
title: "Protocol for Transposed Transactions over HTTP"
docname: draft-unknown-ptth-ptth-latest
category: std
wg: httpbis
ipr: trust200902
keyword: internet-draft
stand_alone: yes
pi: [toc, sortrefs, symrefs]
author:
 -
    fullname: Kazuho Oku
    organization: Fastly
    email: kazuhooku@gmail.com
normative:
  HTTP-SEMANTICS: RFC9110

--- abstract

This document specifies the Protocol for Transposed Transactions over HTTP
(PTTH), an HTTP extension that allows a backend server to establish an HTTP
connection to a reverse proxy and transpose HTTP request flow. The reverse
proxy then forwards incoming requests to the backend server. This extension
lets backend servers behind restrictive firewalls accept HTTP traffic through
reverse proxies without changing firewall settings and with virtually zero
overhead.


--- middle

# Introduction

In scalable HTTP deployments—such as those using CDNs and dynamic backend server
pools—clients send requests to reverse proxies, which then forward them to
backend servers. Backend servers frequently reside behind firewalls that block
inbound TCP or QUIC connections, requiring special network or firewall
configuration to permit proxy-initiated traffic. To overcome these restrictions,
some organizations use VPNs, but VPNs introduce operational complexity, hamper
scalability, and impose performance overhead.

PTTH enables a backend server to establish an HTTP connection to a reverse proxy
and transpose the flow of HTTP requests so that the proxy sends incoming
requests back over the same connection. An HTTP request is used for
authenticating the backend server and for negotiating the scope of requests
forwarded to the transposed connection, providing flexibility to deployments.

Because PTTH transposes the direction of communication rather than
encapsulating traffic, it incurs virtually zero overhead and delivers high
efficiency.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# The Protocol

To set up a transposed connection, the backend server connects to the reverse
proxy and sends an HTTP request including a URI specifying the transposed
endpoint and credentials that authenticate the backend server.

The exact form of the URI specifying the transposed endpoint is unspecified; it
is up to each reverse proxy deployment.

Similarly, the authentication scheme is unspecified. Deployments can use either
a TLS- or an HTTP-based authentication scheme, or something else.

The method being used to establish the transposed connection is different
between HTTP/1.1 and HTTP/3, while the header fields are agnostic to the HTTP
versions being used.

PTTH cannot originate over HTTP/2. To establish a transposed HTTP/2 channel,
HTTP/1.1 upgrade is used, with the ALPN specifying HTTP/2.


## HTTP/1.1

In HTTP/1.1 ({{HTTP1=RFC9112}}), the HTTP upgrade mechanism
({{HTTP-SEMANTICS}} Section 7.8) is used.

The method of the issued request SHALL be "GET", accompanied by an
"Upgrade: ptth" header field.

The request MUST also include the ALPN header field ({{ALPN-HEADER=RFC7639}})
specifying the HTTP versions that the backend server is willing to use on the
transposed connection.

Once the transposed connection is established successfully, the reverse proxy
responds with a 101 (Switching Protocols) response, alongside an ALPN response
header specifying the HTTP version being chosen. After a 101 response is sent,
HTTP requests are sent in the direction from the reverse proxy to the backend
server.

{{fig-tunnel-establishment}} shows an exchange of HTTP/1.1 request and response
establishing the transposed connection. In this example, the Basic HTTP
Authentication Scheme {{?BASIC-AUTH=RFC7617}} is used to authenticate the
backend server. As for the application protocol to be used on the transposed
connection, the backend server is offering both HTTP/2 and HTTP/1.1, and the
reverse proxy selects HTTP/2.

~~~
GET /reverse-endpoint HTTP/1.1
Host: example.com
Connection: upgrade
Upgrade: ptth
Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ==
ALPN: h2, http%2F1.1

HTTP/1.1 101 Switching Protocols
Connection: upgrade
Upgrade: ptth
ALPN: h2

~~~
{: #fig-tunnel-establishment title="Establishing a transposed connection over HTTP/1.1"}

As the parameters for the transposed connection are exchanged using the upgrade
request, they cannot be changed once the transposed connection is established.
To change those parameters, a new HTTP/1.1 connection should be established and
transposed.


## HTTP/3

In HTTP/3 ({{HTTP3=RFC9114}}), the OPTIONS method
({{Section 9.3.7 of HTTP-SEMANTICS}}) is used to transpose HTTP request flow on
the HTTP/3 connection. As the flow of the existing connection is transposed,
neither the `:protocol` pseudo-header field nor the ALPN header field is used.

Once the reverse proxy responds with a 2xx response, it starts forwarding HTTP
requests on the server-initiated, bidirectional QUIC streams. Note that, due to
packet reordering, backend servers might receive these requests before receiving
a 200 response for the OPTIONS request.

Similarly to when HTTP/1.1 is used, establishment of a new HTTP/3 connection is
required when a transposed HTTP/3 connection with different set of parameters is
needed. Once the transposed connection is established, the reverse proxy SHOULD
reset incoming requests that it receives using a H3_REQUEST_REJECTED error
({{Section 8.1 of HTTP3}}).

TODO: Discuss the downsides of transposing an HTTP/3 connection. Kazuho's
understandings are:
* SETTINGS is fine; none of the HTTP/3 settings are specific to clients or
  servers.
* QPACK is fine; One set of QPACK streams can handle requests flying in both
  directions.
* The design does not interfere with WebTransport over HTTP/3; for both client-
  and server-initiated bidirectional streams, WebTransport streams can be
  identified by their signal vallues (0x41), and if they are associated to
  client- or server-initiated requests can be determined by their Session ID
  (i.e., the stream ID of the CONNECT stream).
* We need to consider how to handle quarter stream IDs of HTTP/3 datagrams;
  but that issue not orthogonal to sending HTTP requests in both directions.
  The issue arises for any design that establishes the QUIC connection in the
  reverse direction. Maybe the answer here is to use
  `stream_id / 4 + (2 << 60)` as the quarter stream IDs for datagrams
  belonging to the transposed requests.
* Rather than using OPTIONS, do we want to use an extended CONNECT? While
  use of OPTIONS might be fine, HTTP requests without a special pseudo-header
  is end-to-end per definition. Using an extended CONNECT is a straightforward
  to constrain the setup of a transposed _connection_ to hop-by-hop.


# IANA Considerations

Once approved, this document will request IANA to register the following entry
to the "HTTP Upgrade Tokens" registry maintained at
<https://www.iana.org/assignments/http-upgrade-tokens>:

Value:
: ptth

Description:
: Establishes a transposed HTTP/1.1 connection.

Expected Version Tokens:
: None

Reference:
: this document


--- back

# Acknowledgments
{:numbered="false"}

TODO.
