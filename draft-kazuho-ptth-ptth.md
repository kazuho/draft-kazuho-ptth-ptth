---
title: "Protocol for Transposed Transactions over HTTP"
docname: draft-kazuho-ptth-ptth-latest
category: std
wg: httpbis
ipr: trust200902
keyword: internet-draft
stand_alone: yes
pi: [toc, sortrefs, symrefs]
author:
 -
    fullname:
      :: 奥 一穂
      ascii: Kazuho Oku
    organization: Fastly
    email: kazuhooku@gmail.com
normative:
  HTTP-SEMANTICS: RFC9110

--- abstract

This document specifies the Protocol for Transposed Transactions over HTTP
(PTTH), an HTTP extension that allows a backend server to establish an HTTP
connection to a reverse proxy and transpose the flow of HTTP requests. The
reverse proxy then sends requests to the backend server over the resulting
transposed channel. This extension lets backend servers behind restrictive
firewalls accept HTTP traffic through reverse proxies without changing firewall
settings and with minimal overhead.


--- middle

# Introduction

In scalable HTTP deployments—such as those using CDNs and dynamic backend server
pools—clients send requests to reverse proxies, which then forward them to
backend servers. Backend servers frequently reside behind firewalls that block
inbound TCP or QUIC connections, requiring special network or firewall
configuration to permit proxy-initiated traffic. To overcome these restrictions,
some organizations use VPNs, but VPNs introduce operational complexity, hamper
scalability, and impose performance overhead.

PTTH lets a backend server establish an HTTP connection to a reverse proxy and
transpose the flow of HTTP requests, so that the reverse proxy can send HTTP
requests to the backend server.

PTTH has the following characteristics:

* **URI-based:** the setup request's target URI identifies the PTTH endpoint
  and scopes which requests the reverse proxy forwards to the backend server.

* **Authorized like any HTTP request:** the backend server asserts its identity
  — with a TLS client certificate, an HTTP authentication scheme, or otherwise —
  and the reverse proxy authenticates and authorizes the request just as it
  would any ordinary HTTP request.

* **Unmodified HTTP on the transposed channel:** PTTH merely sets up the
  transposed channel; HTTP runs over it unmodified, so HTTP extensions can be
  used without modification.

* **Minimal overhead:** the encapsulation that a transposed channel would
  otherwise incur can be avoided, letting the channel use the underlying
  transport directly.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Establishing Transposed HTTP Channels

To establish a transposed HTTP channel, the backend server connects to the
reverse proxy and issues an extended CONNECT request ({{!EXT-CONNECT=RFC8441}}
and {{!EXT-CONNECT-H3=RFC9220}}) — or, in HTTP/1.1, the equivalent HTTP Upgrade
({{HTTP-SEMANTICS}} Section 7.8) — that both authenticates the backend server
and negotiates the transposition. Although the way extended CONNECT is expressed
differs between HTTP versions, the accompanying header fields do not. The
parameters for negotiating PTTH are therefore defined in a version-neutral
manner.

The target URI of the establishment request is constructed from the URI
Template described in {{client-config}}. Besides identifying the PTTH endpoint,
the target URI identifies the backend origin for which the backend server is
registering a transposed channel. Likewise, the authentication scheme is
unspecified: deployments can use a TLS- or an HTTP-based scheme, or something
else.

Once a transposed channel is established, HTTP requests flow from the reverse
proxy to the backend server: the reverse proxy acts as the HTTP client and the
backend server as the HTTP server on the transposed channel.

The HTTP version that carries the extended CONNECT request and the HTTP version
of the transposed channel are independent. The backend server can send an
extended CONNECT request on any version of HTTP and establish a transposed HTTP
channel of any HTTP version.

## Client Configuration {#client-config}

Backend servers are configured to use a reverse proxy with a URI Template
({{!URI-TEMPLATE=RFC6570}}) that describes the target URI of the PTTH establishment
request. The URI Template identifies the reverse proxy and carries the backend
origin for which the backend server is registering a transposed channel.

The following examples show URI Templates for registering PTTH channels:

~~~
https://proxy.example.org/.well-known/ptth/{backend_scheme}/{backend_authority}/
https://proxy.example.org:4443/ptth?s={backend_scheme}&a={backend_authority}
https://proxy.example.org:4443/ptth{?backend_scheme,backend_authority}
~~~
{: #fig-uri-template title="URI Template Examples"}

The following requirements apply to the URI Template:

* The URI Template MUST be a level 3 template or lower.
* The URI Template MUST be in absolute form and MUST include non-empty scheme,
  authority, and path components.
* The path component of the URI Template MUST start with a slash ("/").
* All template variables MUST be within the path or query components of the URI.
* The URI Template MUST contain the variables "backend_scheme" and
  "backend_authority" and MAY contain other variables.
* The URI Template MUST NOT contain any non-ASCII Unicode characters and MUST
  only contain ASCII characters in the range 0x21-0x7E inclusive.
* The URI Template MUST NOT use Reserved Expansion ("+" operator), Fragment
  Expansion ("#" operator), Label Expansion with Dot-Prefix, Path Segment
  Expansion with Slash-Prefix, or Path-Style Parameter Expansion with
  Semicolon-Prefix.

Backend servers SHOULD validate the requirements above. However, a backend
server MAY use a general-purpose URI Template implementation that does not
perform PTTH-specific validation. If a backend server detects that any of the
requirements above are not met by a URI Template, it MUST reject its
configuration and abort the request without sending it to the reverse proxy.

The "backend_scheme" and "backend_authority" variables identify the backend
origin being registered. The "backend_scheme" variable contains the scheme of
the backend origin. The "backend_authority" variable contains the authority of
the backend origin, consisting of a host and, optionally, a port. The authority
MUST NOT contain userinfo. If the authority includes a port, the colon (":")
preceding the port is part of the "backend_authority" value before URI Template
expansion, but is percent-encoded when represented in the resulting target URI.
For example, the backend authority "backend.example.com:8443" is represented as
"backend.example.com%3A8443" in the expanded URI. If the authority does not
include a port, the default port for "backend_scheme" is implied.

Using the terms "scheme", "host", and "port" from {{Section 3 of !URI=RFC3986}}, these variables
adhere to the following format:

~~~ abnf
backend_scheme    = scheme
backend_authority = host [ ":" port ]
~~~
{: #fig-origin-vars title="URI Template Variable Format"}

When sending a PTTH establishment request, the backend server MUST perform URI
Template expansion using "backend_scheme" and "backend_authority" from the
backend origin it is registering.

Backend server implementations that are constrained to configuring only the
reverse proxy host and port MAY attempt to use the following default template:

~~~
https://$PROXY_HOST:$PROXY_PORT/.well-known/ptth/{backend_scheme}/{backend_authority}/
~~~

where $PROXY_HOST and $PROXY_PORT are the configured host and port of the
reverse proxy, respectively. PTTH deployments SHOULD offer service at this
location if they need to interoperate with such backend servers.

## HTTP/1 and HTTP/2

To establish a transposed HTTP/1 or HTTP/2 channel, the backend server issues
extended CONNECT accompanied by the "ptth" token: in HTTP/1.1
({{!HTTP1=RFC9112}}), a "GET" request carrying an "Upgrade: ptth" header field;
in HTTP/2 ({{!HTTP2=RFC9113}}) and HTTP/3 ({{!HTTP3=RFC9114}}), a CONNECT
request carrying "ptth" in the ":protocol" pseudo-header field.

The request MUST also carry the ALPN header field ({{!ALPN-HEADER=RFC7639}})
specifying the HTTP versions that the backend server is willing to use on the
transposed channel.

When the transposition succeeds, the reverse proxy returns a successful response
— a 101 (Switching Protocols) response in HTTP/1.1, or a 2xx (Successful)
response in HTTP/2 and HTTP/3 — carrying an ALPN response header field that
specifies the chosen HTTP version. The transposed channel is then carried
directly over the resulting bidirectional byte stream.

Capsules are not used on a transposed HTTP/1 or HTTP/2 channel: the transposed
HTTP protocol supplies its own framing and can exchange metadata — through
header fields, and in HTTP/2 through control frames — so it already provides
what capsules would. Because HTTP/2 offers richer framing and metadata exchange
than HTTP/1.1, HTTP/2 is RECOMMENDED as the transposed protocol.

{{fig-establishment}} shows an HTTP/1.1 exchange establishing a transposed
channel. Here the Basic HTTP Authentication Scheme {{?BASIC-AUTH=RFC7617}}
authenticates the backend server, which offers both HTTP/2 and HTTP/1.1; the
reverse proxy selects HTTP/2.

~~~
GET /.well-known/ptth/https/backend.example.com/ HTTP/1.1
Host: proxy.example.com
Connection: upgrade
Upgrade: ptth
Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ==
ALPN: h2, http%2F1.1

HTTP/1.1 101 Switching Protocols
Connection: upgrade
Upgrade: ptth
ALPN: h2

~~~
{: #fig-establishment title="Establishing a transposed channel over HTTP/1.1"}


## HTTP/3

HTTP/3 ({{HTTP3}}) runs over QUIC ({{!QUIC=RFC9000}}), whose underlying
transport is UDP. A transposed HTTP/3 channel is therefore established as a new
HTTP/3 connection whose UDP flow is proxied over the setup channel.

To establish it, the backend server issues an extended CONNECT request with a
"ptth-udp" upgrade token, together with an ALPN header field specifying HTTP/3.
The new connection is initiated by the reverse proxy toward the backend server;
the reverse proxy is thus the client-side of the transposed connection as well
as being the HTTP client, and the backend server is the server-side of the
transposed connection as well as being the HTTP server, so the transposed
connection is an ordinary HTTP/3 connection in which no transport or stream
roles are reversed.

The UDP flow carrying the new connection is proxied over the setup channel as
datagrams encapsulated using HTTP/3 Datagrams or capsules
({{!CAPSULE=RFC9297}}), as in Proxying UDP in HTTP ({{!CONNECT-UDP=RFC9298}}).
Therefore, while any version of HTTP can be used as the setup channel, using
HTTP/3 as the setup channel provides the opportunity to use HTTP/3 Datagrams and
avoid head-of-line blocking.

To reduce the encapsulation overhead, extensions that optimize the proxying of
UDP MAY also be used; see {{fwd}}.

For the QUIC handshake ({{?QUIC-TLS=RFC9001}}) of the new connection, an
external PSK ({{Section 2.2 of !TLS13=RFC8446}}) is used, which both endpoints
derive from the setup channel's TLS or QUIC connection using the exporter
interface ({{Section 7.5 of TLS13}}), with the label "ptth-udp", an empty
context, and an output length equal to the size of the hash of the cipher suite
negotiated on the setup channel. That hash is also the PSK's associated hash
function, and the key is offered under the PSK identity "ptth-udp".

The purpose of this PSK is not to authenticate the backend server. Instead, it
is used to let the QUIC connection, an always-encrypted transport, inherit the
security context of the setup channel. When the reverse proxy returns a
successful response to the extended CONNECT request, it has already validated
the identity of the backend server as with ordinary HTTP requests, and has
agreed to forward requests. Authentication of the backend server is unneeded
after that point.


# Avoiding Encapsulation Overhead

The transposed channel established as described above is carried within the
setup connection: over HTTP/2 it is confined to a single stream, and over HTTP/3
the new connection's packets are wrapped as HTTP Datagrams and thereby encrypted
twice. This section describes, for each case, how that encapsulation can be
avoided so that the transposed channel uses the underlying transport directly.


## HTTP/1 and HTTP/2

Extended CONNECT over HTTP/2 or HTTP/3 establishes the transposed channel within
a single bidirectional stream. A transposed HTTP/2 channel carried this way is
multiplexed inside that one stream, adding a layer of framing and confining the
transposed channel to that stream's flow-control window.

Performing the setup over HTTP/1.1 avoids this. The HTTP/1.1 Upgrade hands over
the entire connection rather than a single stream, so the transposed channel is
the connection itself; a transposed HTTP/2 channel then runs natively, with its
streams mapped directly onto the connection.

Because the transposed channel reuses the setup connection, it also inherits
that connection's TLS session: its authentication and encryption carry over
unchanged, and neither a pre-shared key nor any additional authentication is
required.


## HTTP/3 {#fwd}

Proxying the transposed connection's packets as HTTP/3 Datagrams or capsules
encrypts each packet twice: once by the transposed connection and again by the
setup channel. Forwarded mode of QUIC-Aware Proxying
({{?QUIC-PROXY=I-D.ietf-masque-quic-proxy}}) removes the second encryption.

In forwarded mode, short header packets of the transposed connection are sent
over the same path as the setup channel but are not encapsulated within it; the
reverse proxy and the backend server identify them by their QUIC connection IDs.
Packet transforms ({{Section 4.3 of QUIC-PROXY}}) are unnecessary in PTTH,
because the reverse proxy is itself an endpoint of the transposed connection;
there is no additional link that passive attackers might observe to correlate.

Endpoints can therefore send and receive the QUIC packets of the transposed
connection directly on the network path of the setup connection, simply by
swapping the Connection IDs to the Virtual Connection IDs assigned by the
forwarded mode.


# Security Considerations


## Establishing Authority

In HTTP, only the URI's authority may process or delegate the request
({{Section 17.1 of HTTP-SEMANTICS}}).

This authority model of HTTP remains unchanged under PTTH:

* When the backend server connects to the reverse proxy and requests the
  transposition of the connection, the backend identifies the reverse proxy
  using a target URI whose authority component identifies the reverse proxy.
* When the reverse proxy forwards requests to the backend over a transposed
  connection, it is merely exercising its rights as the authoritative server.
  This behavior is identical to forwarding requests over connections the
  reverse proxy initiated, using whatever authentication scheme it chooses.
  PTTH differs only in how the backend connections are established.

Accepting a PTTH establishment request does more than authenticate the backend
server. It authorizes that backend server to receive requests for the backend
origin identified by the request target. A reverse proxy MUST therefore verify
that the authenticated backend server is authorized to register the requested
backend origin before accepting the request.


# IANA Considerations

## HTTP Upgrade Token

Once approved, this document will request IANA to register the following entries
to the "HTTP Upgrade Tokens" registry maintained at
<https://www.iana.org/assignments/http-upgrade-tokens>:

Value:
: ptth

Description:
: Establishes a transposed HTTP channel that runs over a byte stream.

Expected Version Tokens:
: None

Reference:
: this document

Value:
: ptth-udp

Description:
: Establishes a transposed HTTP channel that runs over UDP.

Expected Version Tokens:
: None

Reference:
: this document

## TLS Exporter Label

Once approved, this document will request IANA to register the following entry in the "TLS
Exporter Labels" registry maintained at
<https://www.iana.org/assignments/tls-parameters>:

Value:
: ptth-udp

DTLS-OK:
: N

Recommended:
: Y

Reference:
: this document

## Well-Known URI

Once approved, this document will request IANA to register the
following entry in the "Well-Known URIs" registry maintained at <https://www.iana.org/assignments/well-known-uris>:

URI Suffix:
: ptth

Reference:
: this document

Status:
: permanent

Change Controller:
: IETF

--- back

# Acknowledgments
{:numbered="false"}

TODO.
