---
title: "Forward and Reverse HTTP/3 over WebTransport"
abbrev: "HTTP/3 over WebTransport"
category: std

docname: draft-various-httpbis-h3-webtrans-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: ""
workgroup: "HTTP"
keyword:
venue:
  group: "HTTP"
  type: ""
  mail: "ietf-http-wg@w3.org"
  arch: "https://lists.w3.org/Archives/Public/ietf-http-wg/"
  github: "bemasc/h3-webtrans"
  latest: "https://bemasc.github.io/h3-webtrans/draft-various-httpbis-h3-webtrans.html"

author:
 -
    fullname: Benjamin M. Schwartz
    organization: Meta Platforms, Inc.
    email: ietf@bemasc.net
 -
    ins: "Y. Rosomakho"
    fullname: Yaroslav Rosomakho
    organization: Zscaler
    email: yrosomakho@zscaler.com

normative:
  H3:
    =: RFC9114
    display: HTTP/3

informative:
  CAPABILITY:
    title: Good Practices for Capability URLs
    date: 18 February 2014
    target: https://www.w3.org/TR/capability-urls/
  H2:
    =: RFC9113
    display: HTTP/2

--- abstract

HTTP/3 was initially specified only for use with the QUIC version 1 transport protocol.  This specification defines how to use HTTP/3 over a WebTransport session, which can be implemented using any WebTransport protocol.  This enables operation of HTTP/3 when UDP based transport is not available as well as server-initiated HTTP/3 requests.

--- middle

# Introduction

HTTP versions 2 {{H2}} and earlier were primarily specified to run over a reliable stream transport.  Initially, this transport was normally TCP, but TLS over TCP {{?TLS13=RFC8446}} is now often used instead.  Other reliable stream transports are also used, especially for interprocess HTTP requests on a single host.

HTTP version 3 {{H3}} was specified only to run on QUIC version 1 {{!QUIC=RFC9000}}, but its specification anticipates support for other transports ({{H3, Section 3.2}}):

> The use of other QUIC transport versions with HTTP/3 MAY be defined by future specifications.

In fact, nothing in HTTP/3 relies on the transport being a QUIC version at all, so long as it provides transport capabilities similar to QUICv1.  These include:

* A session establishment procedure.
* Support for multiple streams within a session.
* Unidirectional and bidirectional streams, initiated by either party.
* Transmission of independent datagrams (for HTTP/3 Datagrams {{!HTTP-DGRAM=RFC9297}}).

Another transport system that provides these capabilities is the WebTransport session interface {{!WEBTRANS=I-D.ietf-webtrans-overview}}, which we refer to here as "WebTransport".  WebTransport can be implemented within an HTTP/2 {{!WEBTRANS-H2=I-D.ietf-webtrans-http2}} or HTTP/3 {{!WEBTRANS-H3=I-D.ietf-webtrans-http3}} connection, and implementations based on WebSocket and other protocols have been proposed.  A WebTransport server endpoint can always be identified by a URI, which might or might not use the "https" URI scheme.

After a WebTransport session is established, the interface presented to the client and server are largely identical.  Either party can open or accept new streams of either type, send datagrams, and eventually terminate the session.  Thus, once we have defined HTTP/3 over WebTransport (H3-WT), it is straightforward to define Reverse H3-WT, in which the HTTP client and server roles are reversed.

H3-WT is a general-purpose specification that may be put to various uses.  One motivating use case for Forward H3-WT is to enable HTTP/3 over a TCP-based WebTransport protocol, for cases where the client and server prefer HTTP/3 but a network element only permits TCP.  Other creative uses are also possible.

For Reverse H3-WT, one motivating use case is to enable a hidden backend server to delegate TCP server functions to a proxy server.  With this specification, the hidden backend can create a Reverse H3-WT session over which the proxy can issue multiple HTTP CONNECT requests.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

## Roles

To distinguish actions on the WebTransport session from actions on the H3-WT connection that it carries, we define the following roles:

* Dialer - initiates the WebTransport session
* Listener - accepts the WebTransport session
* Client - sends requests on the H3-WT session
* Server - receives these HTTP requests
* Participant - A Client or Server.

In Forward H3-WT, the Dialer is the Client.  In Reverse H3-WT, the Dialer is the Server.

# Specification

## Establishment

Establishment of an H3-WT session is not constrained by this specification.  Any new WebTransport session MAY be used with this specification, subject to prior arrangement by the endpoints.  This specification does not constrain any URLs or protocol IDs associated with the session establishment process.

A future specification could allocate TLS ALPN IDs that indicate the use of this specification with a particular WebTransport protocol.

## Applicability

An H3-WT connection can carry HTTP requests with any method, scheme, authority, and path.  These values are not constrained by any similar values that may have applied to the WebTransport session itself.  For example, if the Listener endpoint is "https://my-origin.example.com/wt", this does not limit the origins to which requests may be sent on the H3-WT connection.

Clients MUST determine the permissible set of origins for an H3-WT session by private arrangement.  Servers SHOULD send an ORIGIN frame {{!ORIGIN=RFC9412}} at the beginning of the connection to indicate which origins are actually available on the session, unless that set is unambiguous (i.e., fixed by private arrangement) or unbounded (e.g., in the case of a proxy service).

## Streams

To create a stream, participants use WebTransport's "create a unidirectional stream" or "create a bidirectional stream" operation.  If this operation fails, it MUST produce a connection error of type H3_STREAM_CREATION_ERROR.

To accept a stream, participants use WebTransport's "receive a unidirectional stream" and "receive a bidirectional stream" operations.  Participants MUST accept additional unidirectional streams whenever there are fewer than 3 active, and SHOULD accept additional bidirectional streams whenever there are fewer than 100.

If the Client receives a bidirectional stream, and no HTTP extension has been negotiated to permit this stream, it MUST produce a connection error of type H3_STREAM_CREATION_ERROR.

### Stream IDs

WebTransport streams do not expose a Stream ID.  To enable HTTP/3 functions that rely on Stream IDs, such as GOAWAY  ({{H3, Section 7.2.6}}) and Datagrams, the creator of each stream MUST write the H3-WT Stream ID at the beginning of each stream, and the H3-WT receiver must consume this Stream ID before passing the stream to HTTP/3.

The H3-WT Stream ID is encoded as a QUIC variable-length integer ({{!QUIC=RFC9000, Section 16}}) and follows the QUICv1 stream numbering convention ({{!QUIC, Section 2.1}}), so the Client's first stream has stream ID 0x00.

## Datagrams

If SETTINGS_H3_DATAGRAM was negotiated on the H3-WT connection, participants send and receive each HTTP/3 Datagram using the WebTransport "send a datagram" and "receive a datagram" operations.  HTTP/3 MUST derive the datagram's Quarter Stream ID from the corresponding stream's H3-WT Stream ID.

Note that datagrams in H3-WT may be unreliable even when H3-WT is running over a reliable protocol, as the HTTP request or the WebTransport session may be forwarded by an intermediary onto a connection that uses a different protocol.

## Closure and Errors

When closing a stream successfully, participants use the WebTransport "send bytes" operation with a FIN indication.  Receipt of a FIN indication in "receive bytes" indicates successful completion of the stream.

When closing a stream abruptly, the stream error code is passed to the WebTransport "abort send side" or "abort receive side" operation as appropriate, and retrieved by the other participant from the "send side aborted" or "receive side aborted" event.

An H3-WT connection is terminated using the WebTransport "terminate a session" operation.  The connection error is passed to this operation and retrieved by the other participant from the "session terminated" event.

# Security Considerations

HTTP clients generally rely on the transport (TCP, TLS, or QUIC) to ensure that they are connected to the intended server before sending a request.  For "https" URIs, this involves TLS server authentication.  In H3-WT, the implementor is responsible for employing suitable authentication of the WebTransport session.  WebTransport guarantees TLS-equivalent authentication of the Listener to the Dialer, which may be sufficient for some Forward H3-WT deployments.  Authentication of the Dialer to the Listener can be accomplished using TLS client authentication (if exposed by the WebTransport protocol), HTTP Authentication (when using WebTransport over HTTP), or Capability URLs {{CAPABILITY}} (when using WebTransport with a URI scheme that supports a non-empty path).

If the WebTransport protocol exposes a TLS Exported Authenticator capability, participants MAY use it to enable Secondary Certificate Authentication {{?I-D.ietf-httpbis-secondary-server-certs}} and/or HTTP Concealed Authentication {{?RFC9729}} within the H3-WT session.

# Examples

## Hidden Origin Configuration

In some cases, it can be desirable for an HTTP origin server to operate through a public gateway without exposing a publicly reachable IP address.  For example, the origin server might be subject to request flood attacks if its IP address were publicly reachable.  Reverse H3-WT allows a hidden origin server to make itself available only to the gateway.

One potential implementation proceeds as follows:

1. The public gateway instructs the origin operator to use "https://gateway.example/reverse/$customer_id" as the URL for Reverse H3-WT, along with a specified secret bearer token.
1. The origin operator selects a generic HTTP gateway implementation that supports Reverse H3-WT over HTTP with bearer authentication, and configures it to forward requests to the origin server.
1. The origin operator configures their local gateway with the specified URL and bearer token.
1. The local gateway opens a WebTransport session to the public gateway using the specified URL using an Extended CONNECT request.
1. The public gateway verifies that the Extended CONNECT request carries the correct bearer token for this customer ID.
1. The local gateway uses the ORIGIN frame to enumerate the origins that are available on this session.
1. The public gateway validates that this customer is permitted to serve the indicated origins.
1. The public gateway starts forwarding incoming HTTP requests for those origins over this session.

~~~aasvg
                        .---------------.
                        | Origin Server |
                        `---------------`
                                ^
                                |                Private
                         .------+------.         Network
                        | Local Gateway |
                         `------+------`
                                |
                    ┄┄┄┄┄┄┄┄┄┄┄┄.┄┄┄┄┄┄┄┄┄┄┄  Firewall
                                |
                                v
                         .--------------.
                        | Public Gateway |
                         `--------------`
                                ^                Public
                                |                Internet
                        .-------+--------.
                        |  HTTP Client   |
                        `----------------`
~~~
{: title="Hidden Origin Configuration"}

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge authors of previous reverse HTTP drafts.
