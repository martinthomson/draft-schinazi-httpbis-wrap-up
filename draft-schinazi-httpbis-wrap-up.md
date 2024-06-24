---
title: The HTTP Wrap Up Capsule
docname: draft-schinazi-httpbis-wrap-up-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
category: std
wg: HTTPBIS
area: "Web and Internet Transport"
venue:
  group: "HTTP"
  type: "Working Group"
  home: https://httpwg.org/
  mail: "ietf-http-wg@w3.org"
  arch: "https://lists.w3.org/Archives/Public/ietf-http-wg/"
  repo: https://github.com/DavidSchinazi/draft-schinazi-httpbis-wrap-up
  latest: https://davidschinazi.github.io/draft-schinazi-httpbis-wrap-up/#go.draft-schinazi-httpbis-wrap-up.html
keyword:
  - secure
  - tunnels
  - masque
  - http-ng
author:
  -
    ins: D. Schinazi
    name: David Schinazi
    org: Google LLC
    street: 1600 Amphitheatre Parkway
    city: Mountain View
    region: CA
    code: 94043
    country: United States of America
    email: dschinazi.ietf@gmail.com


normative:

informative:
  H2:
    =: RFC9113
    display: HTTP/2
  H3:
    =: RFC9114
    display: HTTP/3


--- abstract

HTTP intermediaries can provide a variety of benefits to HTTP systems, such as
load balancing, caching, and privacy improvements. Deployments of
intermediaries also need to be maintained, which can sometimes require taking
intermediaries temporarily offline. In order for this maintenance to avoid
disrupting communications, multiplexed versions of HTTP leverage the GOAWAY
frame to instruct clients to send future requests to a different intermediary
without impacting their active requests. However, the GOAWAY frame is only
available when the intermediary operates inside the encryption that protects
HTTP: gateways can leverage it but CONNECT and connect-udp proxies cannot,
because they cannot observe or modify a tunneled HTTP connection that flows
through them to inject a GOAWAY frame. This document specifies a new "WRAP_UP"
capsule that allows a proxy to instruct a client that it should not start new
requests on a tunneled connection, while still allowing it to finish existing
requests.

--- middle

# Introduction

HTTP intermediaries (see {{Section 3.7 of !HTTP=RFC9110}}) can provide a
variety of benefits to HTTP systems, such as load balancing, caching, and
privacy improvements. Deployments of intermediaries also need to be maintained,
which can sometimes require taking intermediaries temporarily offline. In order
for this maintenance to avoid disrupting communications, multiplexed versions
of HTTP leverage the GOAWAY frame (see {{Section 5.2 of H3}} and {{Section 6.8
of H2}}) to instruct clients to send future requests to a different
intermediary without impacting their active requests. This works well in
situations where the gateway participates in both HTTP connections; i.e., when
the first connection is between the client and gateway and the second is
between the gateway and origin.

~~~ aasvg
+--------+      +---------+      +--------+
| Client |      | Gateway |      | Origin |
|        |      |       * |      |      * |
|        +======+ GOAWAY| +~~~~~~+      | |
|      <----------------+               | |
|                         HTTP Responses| |
|      <--------------------------------+ |
|        +======+         +~~~~~~+        |
+--------+  ^   +---------+   ^  +--------+
            |                 |
     TLS --'                   '-- TLS
~~~
{: #diagram-gateway title="Gateway Sends GOAWAY"}

However, it doesn't when one connection is tunneled over the other; i.e., when
the first HTTP connection is between the client and proxy and the second is
between the client and origin. This happens when the client sends a CONNECT
(see {{Section 9.3.6 of HTTP}}) or connect-udp (see {{?CONNECT-UDP=RFC9298}})
request to the proxy, and then uses the newly established tunnel as the
underlying transport to then establish a second HTTP connection directly to the
origin. When HTTPS is in use on this second connection, the proxy does not have
access to the cryptographic secrets of the client-origin connection, and is
therefore unable to inject a GOAWAY frame. The proxy therefore needs a new
mechanism to indicate graceful termination to the client.

~~~ aasvg
+--------+      +---------+      +--------+
| Client |      |  Proxy  |      | Origin |
|        |      |       * |      |      * |
|        +======+WRAP_UP| |      |      | |
|      <----------------+ |      |      | |
|        +~~~~~~~~~~~~~~~~+~~~~~~+      | |
|                         HTTP Responses| |
|      <--------------------------------+ |
|        +~~~~~~+~~~~~~~~~+~~~~~~+        |
|        +======+         |   ^  +        |
+--------+  ^   +---------+   |  +--------+
            |                 |
     TLS --'                   '-- TLS
~~~
{: #diagram-proxy title="Proxy Sends WRAP_UP"}

This document specifies a new "WRAP_UP" capsule (see {{Section 3 of
!HTTP-DGRAM=RFC9297}}) that allows a proxy to instruct a client that it should
not start new requests on a tunneled connection, while still allowing it to
finish existing requests.

## Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the terms "connection", "client", and "server" from
{{Section 3.3 of HTTP}} and the terms "intermediary", "proxy", and "gateway"
from {{Section 3.7 of HTTP}}.

# Mechanism

This document defines the "WRAP_UP" capsule (see {{iana}} for the value). The
WRAP_UP capsule allows a proxy to indicate to a client that it wishes to close
the request stream that the capsule was sent on. The WRAP_UP capsule has no
Capsule Value.

## Proxy Behavior

Proxies often know in advance that they will close a request stream. This can
be due to maintenance of the proxy itself, load balancing, or applying data
usage limits on a particular stream. When a proxy has advance notice that it
will soon close a request stream, it MAY send a WRAP_UP capsule to share that
information with the client.

## Client Handling

When a client receives a WRAP_UP capsule on a request stream, it SHOULD try to
wrap up its use of that stream. For example, if the stream carried a
connect-udp request and is used as the underlying transport for an HTTP/3
connection, then after receiving a WRAP_UP capsule the client SHOULD NOT send
any new requests on the proxied HTTP/3 connection - but existing in-progress
proxied requests are not affected by the WRAP_UP capsule.

## Minutiae

Clients MUST NOT send the WRAP_UP capsule. If a server receives a WRAP_UP
capsule, it MUST abort the corresponding request stream. Endpoints MUST NOT
send the WRAP_UP capsule with a non-zero Capsule Length. If an endpoint
receives a WRAP_UP capsule with a non-zero Capsule Length, it MUST abort the
corresponding request stream. Proxies MUST NOT send more than one WRAP_UP
capsule on a given stream. If a client receives a second WRAP_UP capsule on a
given request stream, it MUST abort the stream.

Endpoints that abort the request stream due to an unexpected or malformed
WRAP_UP capsule received over HTTP/3 SHOULD use the H3_DATAGRAM_ERROR error
code when aborting the stream.

# Security Considerations

While it might be tempting for clients to implement the WRAP_UP capsule by
treating it as if they had received a GOAWAY inside the encryption of the
end-to-end connection, doing so has security implications. GOAWAY carries
semantics around which requests have been acted on, and those can have security
implications. Since WRAP_UP is sent by a proxy outside of the end-to-end
encryption, it cannot be trusted to ascertain whether any requests were handled
by the origin. Because of this, client implementations can only use receipt of
WRAP_UP as a hint and MUST NOT use it to make determinations on whether any
requests were handled by the origin or not.

# IANA Considerations {#iana}

This document (if approved) will request IANA to add the following entry to the
"HTTP Capsule Types" registry maintained at
<[](https://www.iana.org/assignments/masque)>.

Value:

: 0x272DDA5E (will be changed to a lower value if this document is approved)

Capsule Type:

: WRAP_UP

Status:

: provisional (will be permanent if this document is approved)

Reference:

: This document

Change Controller:

: IETF

Contact:

: ietf-http-wg@w3.org

Notes:

: None
{: spacing="compact" newline="false"}

--- back

# Acknowledgments
{:numbered="false"}

This mechanism was inspired by the GOAWAY frame from HTTP/2.
